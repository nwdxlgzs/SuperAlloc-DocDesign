# 并发分配、所有权与回收

本文是实现前的设计合同，不代表已有代码或性能结果。SuperAlloc 只提供显式 arena 的独立函数，可同时运行多个 arena；不替换进程的 malloc、new 或 mmap。接口的状态码、输出参数及生命周期规则以 [API 设计](api-contracts.md) 为准。

## 1. 普通模式的数据路径

### 1.1 分层与默认配置

- **Arena**：拥有连续地址范围、统一 extent broker、元数据池、线程 context 池和预算。不同 arena 不共享可变分配状态。
- **Thread context / logical heap**：显式登记，运行时只由一个线程操作其私有状态；线程在多个 arena 中工作需要分别登记。普通远程释放可以使用目标 arena 中当前线程的有效 context。
- **Shared fallback heap**：每个 arena 在创建时预建；普通无 context 调用通过其锁串行访问共享状态，不隐式登记线程、不借用外部 heap。它提供便利合同，不宣称与显式 context 路径同等扩展性。共享 heap 的 slab 仍遵守单个有效 owner lease。
- **Slab**：只容纳一个 size class，由一个 heap 独占其本地 freelist、分配状态和 live 计数。远程线程只写该 slab 的远程归还入口。
- **Extent broker**：统一分配 slab、大对象、页映射、RT pool 和内部 metadata；普通对象快路径不访问 broker。

第一版采用下列可测量基线，后续只能根据基准和 ADR 调整：

| 项目 | 初始默认值 / 规则 |
| --- | --- |
| malloc 保证对齐 | 至少 16 B；更大对齐走显式 aligned 接口 |
| slab 目标大小 | `max(64 KiB, OS page size)`；descriptor 为区域内侧置元数据 |
| slab 服务请求上限 | 32 KiB；超过上限或无法满足对齐要求则申请独立 extent |
| 小 size classes | 16 B 到 256 B，每 16 B 一档 |
| 其余 size classes | 对 B = 256、512、…、16384 B，增加 `5B/4、3B/2、7B/4、2B` 四档，共 44 档 |
| 远程 drain 预算 D | 普通一次慢路径总计最多处理 64 个节点，可在创建配置中调整为正整数；显式维护另受调用者三维总预算限制 |
| fast path OS 调用 | 0；实际缺页仍取决于 backing 和驻留状态 |

类表只承担正常请求的向上取整，不能改变用户请求长度、calloc 清零长度或 realloc 保留长度的接口合同。默认 64 KiB slab 的 32 KiB 类有两个 payload slot，slab descriptor 不从这两个 slot 中扣除；全部侧置元数据仍计入 arena 预算。

S-01将SlabDesc设为192B分区记录，ctx+heap模型成本为64位1664B、32位960B，不含payload slab。普通小对象默认不保存逐slot请求长度；精确requested live需启用额外u16侧表，未启用时标为不可用，不能把slot容量作为请求量。详见[布局](metadata-layout.md)与[预算](metadata-budget.md)。

### 1.2 分配和本地释放

1. context 选择 size class 的 `active_slab`；从 owner 私有 freelist 取出 slot，增加 owner-only `live_count`。成功的普通私有路径不更新 arena 全局统计原子或 quota。
2. freelist 为空时，在此次慢路径剩余 D 预算内处理当前 slab 的归还；仍不足则检查该类 `owned_ring_head` 指向的一个不同候选，并推进环首。候选存在本地空位或经剩余预算接收归还后可作为新的 active slab；仍不足才请求 central/refill。这个少量候选规则属于通常路径，不构成普通模式最终失败前的搜索上限。
3. 大对象或特殊对齐请求直接进入 extent 慢路径；其释放需要的 descriptor 和范围记录必须在分配时预备，不能到 free 时才申请可能失败的元数据。
4. 同 owner 的 free 可直接放回本地 freelist并减少 `live_count`。除非满足完整 slab 退休条件，否则不销毁 descriptor。

普通模式的 malloc 可以触发锁、commit、映射或缺页，不承诺严格步骤界限。普通模式的合法 free 不因“归还队列满”失败；它可能进入普通慢路径。recoverable arena 的释放成功点另见 [持久化](persistence.md)，不能用普通远程入队代替 durable commit。

recoverable free 在提交被确认前，不能把 payload 改写为 remote next/freelist、清零或 discard backing；否则崩溃后账本可能仍恢复该对象为 live，而 payload 已被破坏。待提交归还记录必须侧置，只有持久提交确认后才进入普通空间回收。因此下文的 intrusive remote 链不能直接充当 durable 提交前队列。

`owned_ring_head` 所在环包含该 heap/size class 的全部 slab，包括 full、partial 和 active，使用既有 `class_prev/class_next` 双向环字段；`active_slab` 只是环中节点的缓存引用，不构成第二份所有权或另一条链。完整 slab 的 remote head 也可能积累可归还对象，不能因它没有本地空位就把它从维护集合移除。

central/refill无法满足请求时，既定一次回收/重搜必须检查当前heap相关class整环中**已经本地可用**的slot，不能因通常路径只看一个候选就遗漏现成空位并错误报告容量不足。该扫描持有效owner lease或shared锁，期间不新增/摘除环节点，一次有穷遍历可使用受此lease保护的起始节点作局部哨兵；不需要新记录或全局游标。remote drain仍共用此次慢路径剩余D，不在遍历每个slab时重置。普通模式不承诺该最终搜索有固定小步数，不能将它放进RT；仅在remote或休眠其他owner中的容量仍按当前不可复用状态与真实额度报告失败。

## 2. 远程 free 的发布协议

### 2.1 MPSC 入口与 owner backlog

每个 slab 有一个原生指针宽的 atomic remote head。归还链的 next 可放在已释放 slot 的 payload 中；16 B 最小 slot 足以存放 32/64 位指针。只有合法释放者可以写自己的归还节点。

- 生产者 relaxed 读取 head，把该值写入自身节点的 next，再以 success-release/failure-relaxed 的 strong CAS 尝试把自身节点发布为新 head。竞争失败时只重写自己的 next 并重试；首版不依赖 weak CAS 伪失败的隐含公平性。
- 生产者不解引用观察到的旧 head，也不读取旧 head 节点的 next。
- CAS 成功是此次 free 对 slot、slab descriptor 的**最后一次访问**。指针验证、返回值与错误输出准备、sanitizer annotation、poison、依赖 descriptor 的统计采样均在发布前完成；发布后只能处理当前生产者独占的 context/栈收尾，不能为了报告错误、记录计数或调试再访问归还内存。
- owner 仅在 backlog 为空时通过 acquire exchange 把 remote head 置空并接收整链，再保存为 owner 私有 backlog。不为计算长度、尾节点或拼接链而遍历全部节点；每弹出一个节点消耗一次对象预算。
- backlog 未空时先处理它；仅在 backlog 为空时取新 remote 链。这样每次维护的用户态遍历量有明确预算。

普通一次分配慢路径的 D 是全部访问 slab 合计的 drain 上限，不能在每个候选重新获得 D。显式维护按照第4.3节的 `max_objects/max_pages/max_steps` 总额度推进，可处理超过 D 的多个维护项；这不改变普通慢路径的上限。

该入口不等待 owner 运行，也没有需要扩容的独立队列节点。它不代表每个生产者都会及时完成：strong CAS 失败表明观察区间内 head 被其他操作改变，其他生产者或 consumer 可以持续改变 head，使一个生产者一直重试。原生原子 lock-free、系统级活动和单线程 starvation freedom 是三种不同保证；普通 alloc/free 整个 API 仍包含锁和普通慢路径，不能总称为 lock-free。RT 使用 [独立的 slot 状态协议](realtime.md)。

owner 在 exchange 后、写入 backlog 前暂停时，链暂存于该执行中调用的局部状态，由同一个 owner lease 保护。维护线程不能在此时认定链丢失并接管；detach 必须等待此调用完成。异步杀死一个正在操作 allocator 的线程不在支持的恢复合同内。

### 2.2 live 计数与回收证明

`live_count` 的含义是“已发给用户、尚未被 owner 完成归还处理的 slot 数”。它包括用户仍持有的对象、正在远程发布的对象、remote 链中的对象和 backlog 中尚未处理的对象。

1. 分配时仅 owner 增加 live。
2. owner 本地 free 在本地完成后减少 live。
3. 远程 free **不减少 live**；owner 接收到并处理对应节点时才减少。
4. 因此，一个尚未成功发布的合法 remote free 仍贡献一个 live，owner 不能提前退休该 slab。
5. 一个已发布的 free 不再访问 descriptor；owner acquire 接收并处理最后一个归还后，才可能达到 live 为零。

slab 逻辑退休要求 owner 独占权限仍有效、`live_count == 0`、remote head 与 backlog 均为空。先停止该 slab 的再次分配并摘除 owned ring/active 引用，再在已有记录上移交退休责任；控制面仍持 pin 时允许保持退休状态，但不能复用 descriptor、payload 或撤掉仍被查询依赖的存储。控制面引用全部结束后撤销对象索引，取得已预备的清理资源，按统一锁顺序归还 extent，最后回收 descriptor。index/PageTile 节点本身按其 quiescent 回收合同处理，不能因一个 slab 为空就释放仍可被读者观察的索引节点。调试构建应校验条件间的一致性，而不是忽略不一致后继续回收。

该证明针对合法的单次 free。重复 free、并发释放同一对象、使用已失效的对象或 arena handle 属于接口违约，不靠生产版全局原子计数兜底。

### 2.3 ABA 与弱内存序

- remote stack 的生产者只推入自己独占的节点，consumer 一次 exchange 整链；生产者不执行 pop、不根据旧 head 的 next 决定新结构。旧 head 地址经历回收后再次成为 head，不会使生产者使用旧 next，因此这种 head 数值 ABA 不等同于 Treiber pop 的丢节点问题。
- 上述结论依赖 live 计数、节点独占和 descriptor lifetime 协议同时成立；不能把此入口改成多消费者 pop 后仍沿用该证明。
- 归还节点 next 的普通写由 release CAS 发布。C++20 中，后续对同一 head 的 RMW 属于从先前 release 开始的 release sequence；owner acquire exchange 读到最后发布值时，也获得该批较早节点的 next 写。运行中 head 只能用 CAS/exchange RMW 修改；初始化的普通 store 必须在 descriptor 暴露前完成，不能插入 relaxed store-reset 破坏该证明。只依赖 x86 的强序是不合格实现；必须在 ARM64 和32位 ARM 目标上验证。
- 陈旧用户指针在同地址再次分配后的 ABA 无法由裸指针通用 API 完整检测。诊断 handle 或 generation 可以帮助定位错误，不改变合法调用合同。

### 2.4 backlog 顺序与有限进展

一次 exchange 得到的批次至多包含该 slab 的 `slot_count` 个不同归还节点。批内按 intrusive 栈顺序处理，后来的 remote 发布留在下一批；只要旧 backlog 未空，就不能再 exchange 一批并插在它前面。这样新流量不能无限插队到已经接收的旧节点之前。

有限批次在 owner 得到持续调度、该 slab 得到持续正数 drain 预算且没有故障冻结时最终处理完毕；这不是墙钟上界。尚未成功 CAS 的生产者还没有进入批次，不能用该论证证明它不饥饿。活 owner 休眠时 remote free 仍能发布，但空间会留在 remote/backlog，直到 owner 协作运行；arena maintain 不会强抢它。

## 3. Owner 身份、线程退出与 arena 生命周期

### 3.1 原生宽 owner token

每个 arena 在登记慢路径中分配单调、不复用的 `uintptr_t` owner token，保留零值等必要控制状态。token 耗尽时返回 `LIMIT` 并拒绝新登记，不允许回绕。该上限约束 arena 生命周期内的**累计登记次数**，与同时活跃 context 的配置上限不同；32 位 Android 也不得因已退出线程很多而复用旧 token。descriptor 的 owner 身份只需一个原生宽原子，不把 context 指针和 generation 强塞入需要双字 CAS 的结构。

context 地址可以在安全回收后复用，但 owner token 不复用，避免旧 descriptor 观察值被误认为新 context 的本地 slab。owner-only 快路径要求当前 context 的 token 匹配且持有有效 ownership lease。同一 context 不能并发使用，也不能因调用的是 page/map 慢路径就绕过这个限制；无 context 入口使用其单独同步合同。

所有实际参加快路径的原子都必须在支持目标上为 lock-free；32 位 Android 不能暗中使用依赖 libatomic 锁的 64 位 CAS。持久格式的 64 位 offset、统计累计值与快路径原子是不同概念，可在慢路径锁内访问或由 owner 局部统计后汇总。

### 3.2 detach 与 orphan 接管

线程应显式 detach；可选 TLS 清理仅作为正常线程退出时的便利机制，不把被异步终止的线程当成可自动安全恢复的情况。

detach 在 context 所属线程的安全点执行：

1. 禁止该 context 发起新分配，完成其当前调用；远程生产者仍可通过稳定 descriptor 归还。
2. 处理允许预算内的归还，把私有 freelist、backlog、live 和 slab 管理状态留在 arena 内；这些状态不能指向退出线程的栈。
3. 在旧 owner lease 下摘除各 slab 的 owned ring/active 引用，完成对 owner-only 字段的最后访问，再通过 central ownership 锁发布 ORPHAN 并排入 `central_tail`。旧 heap 与 central 不能同时声称拥有该 slab；remote head 入口保持稳定。
4. adopter 在 central 锁内独占领取并 pin orphan，随后释放 central 锁。在目标私有 context 的 lease 或 shared heap 锁下接管 backlog/live/freelist、设置 `current_heap`、加入相应 owned ring，最后 release 发布目标 owner token。不能先暴露 shared token，再在未持 shared 锁时初始化 owner 字段。
5. context 已不持有 slab、控制面引用和执行中的调用后，才可把 context 存储归还元数据池。

移交允许 slab 中仍有用户对象，无需搬动 payload。维护线程不能强行窃取仍活跃或只是休眠的 owner 私有 freelist；需要应用协作 safepoint、detach，或等待其下次进入分配器。

central 锁的 release/acquire 衔接“旧 owner 最后写 → adopter 取得 claim”，目标 owner 身份发布衔接完整 heap 状态；读 owner token 判断本地路径时采用匹配的 acquire 读取并验证当前 lease。remote-only 路径不因观察到某个 token 就获得 owner-only 访问权限。arena maintain 把 orphan 领入预建 shared heap 后在 shared 锁下处理，不能临时发明一个没有布局/生命周期的维护 owner。队列及预算规则见第4.3节。

### 3.3 查询与销毁

合法对象的 free 可利用其 live 引用保证 descriptor 生命周期；无对象所有权的统计、遍历、诊断和地址查询不能套用这个保证。它们通过控制面锁、明确的快照 pin 或 quiescent inspection 读取元数据，不能无保护追踪可退休 descriptor。

arena 关闭/销毁要求停止新进入、所有 context 退出执行、所有控制面操作结束。普通 destroy 对仍有有效对象的处理以 API 合同为准，不自动把用户仍持有的指针变成安全可释放对象。不同 arena 的相同 token 数值没有相同身份；所有操作必须携带正确 arena/context。

## 4. Broker 竞争、预算与有界维护

### 4.1 页级操作与分配混用

统一 broker 维护 Metadata、MallocSlab、LargeObject、PageMap、RTPool、Pending、Free 等互斥所有权。malloc 的内部页不能被用户 page-unmap、page-protect 或 fixed mapping 部分覆盖。

跨范围操作遵循：预备 descriptor → 在 broker 锁内验证并标记 Pending → 锁外执行 OS 操作 → 锁内发布成功状态或回滚/隔离失败范围。Pending 范围对其他申请不可用；OS 调用、文件 I/O 和 flush 不持有全局 broker 锁。

首版使用可证明的单 broker 锁，结合 owner slabs 避免普通快路径竞争。只有基准确认页操作或 span 周转使该锁成为瓶颈，才考虑分片；分片必须同时设计跨分片固定映射、多范围回滚和统一锁序，不能只把锁数量增加。

锁外 preflight 得出的需求会在取 broker 锁后复核；计划失配至多进行一次内部重新规划，再失配返回 `BUSY`，此时尚未改变用户范围或调用相应 OS 动作。此前独立完成的 metadata 扩容可保留，不假称整个调用绝无资源变化。该次数限制只防止规划循环无限重启，不界定等锁、CAS 或 OS 调用时间。

所有 slab/范围索引和 pending 状态规则由 [整体架构](architecture.md) 与 [区域内元数据](arena-metadata.md) 定义。malloc 和 RT 的合法 free 定位走预建固定深度索引及 slot 算术，不依赖扫描所有 extent。

锁层级遵循架构文档的完整约束：central/shared heap 锁与 broker 锁禁止嵌套。refill 先独立取得并初始化 extent，再取得 heap 锁发布；退休先在 heap 锁内摘除并 pin，释放该锁后再还 broker。metadata pool 普通锁是叶子锁，broker 只使用预留记录；metadata growth gate 是受约束的独立慢路径。生命周期锁只保护状态切换/准入，journal 提交也不得持有 broker 或 heap 锁执行 I/O。

### 4.2 内存预算

明确区分以下计量：用户请求 live bytes、slot usable bytes、heap private span capacity、cached free bytes、metadata、committed bytes、reserved VA，以及 OS 采样的 resident bytes。

- private capacity 按整 slab 预付额度，避免每次 malloc/free 都更新全局计数。
- 私有容量、空 slab 保留量、context 数量、metadata 和总 commit 均有配置上限。全局预算转移发生在 slab/context 慢路径。
- 带 live 对象的 slab 可在安全点移交 central；稳定 remote 入口承接后续 free，不需要复制用户对象。
- 压力处理使用既有分配慢路径、显式 thread flush/arena maintain 和应用 safepoint；首版不添加未定义的 pressure epoch、广播字段或后台线程。普通 OOM 路径只执行配置允许的一轮有限回收并重新搜索一次，不等待所有休眠线程响应。
- 局部统计可在 owner 内累计；跨线程精确快照需要协调，普通低成本查询应标明采样/近似属性，不能虚构实时全局一致性。

因此，系统能限制预留给私有 heap 的容量，但不能同时无条件保证“每次操作无共享原子”与“任意时刻所有空闲字节立即归还全局”。

### 4.3 公平维护、orphan 与退休责任链

线程显式 flush 使用 `HeapHeader.maintenance_class_cursor`（0..43）和每类 `owned_ring_head` 两层游标；没有额外游标分配。算法如下：

1. 一次 class 检查消耗一个 `max_steps` 单位，空类同样推进 class cursor。非空类仅访问当前环首一个 slab，使用已有 owner lease。
2. 若对象预算充足，对该 slab 至多处理 `min(D, remaining_objects)` 个 backlog/remote 节点，逐个计入本次调用的 `objects`。对象 drain 不消耗 pages；span 移交/退休或后端页动作另按完整工作项计 pages/steps，详见[配置](configuration.md)。
3. 每次已执行的 slab 检查都把该类环首转至下一个仍存活节点，不因“没有归还”“暂不能退休”而固定盯住它；class cursor 随后推进至下一类。slab 摘除时先保存合法 successor；singleton 摘除后环首为空。
4. 剩余 objects 不足以接收一个节点时结束本轮对象扫描，保留下一 class 位置给下次调用；不能耗尽预算后仍转完整圈，令下次总从同一热类开始。steps=0 时不工作；objects=0 的调用不作对象 drain，也不获得相应公平性保证。

普通 demand-driven 通常路径只检查当前类的少量候选，最终申请失败前另有第1.2节的一次有穷相关class重搜；两者都不能替代显式 flush 对冷类的轮转。稳定有限的 owned 集合在持续有足够 steps/objects 的维护下，每个 class/slab 最终得到访问；动态加入/移交/退休、锁竞争和线程暂停下不承诺固定调用轮数或等待时间。调用方预算过小而始终容不下一个完整 span/OS 工作项时，该项可一直待处理，维护结果必须如实提示余留责任。

arena maintain 还在 shared heap 锁下维护**已有shared集合**，使用shared自带的两层游标，每调用至多访问一个class/环首候选，drain至多 `min(D, remaining_objects)`。这样已adopt、暂时没有同类新分配的shared full slab也能最终接收归还；不能仅维护central新来的orphan。它没有取得任何活私有heap的owner权限。

central orphan 和 retired 各使用现有的 `central_tail/retire_tail` 单根及记录内链接构成循环 FIFO；根保存 tail，非空队头为 tail 的 next。新责任追加尾部，不申请队列节点。SlabDesc 的 `retire_next` 在 orphan 队列、退休队列和已领取状态之间互斥复用；RTPool 同样先摘除 LIVE 活动链再进入退休链。终态隔离 RangeTxn 先退出 active Pending 链，再复用 `active_next` 进入 retire 队列；按 `prefix.kind` 选择正确的 next 字段，不能把X当作含 `retire_next` 的记录解读。

隔离范围的既有ExtentDesc在非Free状态使用 `free_prev/free_next` 连接该X的 `cleanup_list`；孤立后端责任仍由原steps记录跟踪，X的 `completed_count` 在终态清理阶段作为已有step数组的扫描位置。队列claim必须同时校验责任归属：同一OS资源只能由X或原owner退休项的一方清理，移交责任前不得把两者同时排成可执行责任。详细转移见[操作协议](operation-protocols.md)与[状态机](state-machines.md)。

每次 arena maintain 对每类队列只领取一个小 batch，设计基线上限各8条；两类合用固定16指针栈数组（64位128B/32位64B，不含普通调用局部变量），不增加arena记录或公共配置。central 锁内从最老项逐条摘除、标记唯一 claim 并 pin，按剩余预算允许少取；释放锁后才取得目标 heap/broker 资源。每条领取及基本检查消耗一个step，移交和OS子动作另消耗其规定的steps/pages；不得因为暂时不能执行而免费遍历。

选择batch时先为两个非空类别各保留一次基本服务机会，再从orphan开始交替取得额外项，且必须给后续实际工作保留相应预算；预算不足时减小batch。处理先后固定为：orphan首项、retire首项、shared一个候选，随后交替处理已领取的其余orphan/retire项。前两项不得耗用给shared一次class检查和一个可接收对象保留的steps/objects；额外batch领取也不能吃掉这三类的已保留工作预算。若某完整span/OS项超过可分配的剩余额度，则跳过并保留责任，不为完成该项突破总budget。

未完成、暂不能处理或剩余预算不足的责任在本batch末回到队尾；一调用每类只开一个batch，绝不重新领取刚返回的失败项。这样即使多个maintainer并发、别的调用摘除/回收了初始tail，也不需要不安全的裸tail哨兵、全队列访问集合或新增visit epoch。claim只在安全完成/物理释放或明确重新排队后结束；同步调用的线程不得被异步终止。

batch只是暂时独占少量清理责任；remote free仍通过稳定入口发布，合法逻辑free不等待这个batch完成，可能推迟的是物理回收。8条上限只是可测设计基线，未证明为最优。arena maintain不等价于“清空全部队列”，调用方需要按维护目标反复调用。

预算持续足够让orphan、retire各一个完整工作项以及shared一次有对象额度的候选访问执行时，才有三类之间的条件性完成进展保证；仅够前一两类时不能宣称后类不饥饿。某类为空时其预留可转给其余实际工作。调用方应提供充足预算；首版不以新增全局轮转原子弥补这一配置限制。FIFO只保证成功得到对应类别服务的已入队记录不被后来追加项插队；锁调度不公平、所有权尚未交接、资源永久不足或后端永久失败仍可阻止完成。失败责任保持归属与费用，不能当作broker Free或自动丢弃。

### 4.4 NUMA 与尾延迟边界

首版使用owner first-touch与显式准备路径的预提交/预触页；通用node preference、严格NUMA放置和自动迁移属于F-01后续验证，不在本版ABI提供未定义的选项。记录实际拓扑与线程绑定，不能从first-touch推断平台保证了指定节点放置。

远程 drain、维护扫描和压力响应均采用显式预算。普通模式仍可能遇到 CAS 竞争、broker 锁、缺页、调度暂停和存储 I/O；不能从“通常不加锁”推导严格实时保证。

单 shared heap、broker、metadata pool/growth、central 和 durable 提交都有实际串行服务上限；高线程数不自动提升这些路径的吞吐。竞争点、积压守恒、退路和测量门槛见[竞争与吞吐设计](contention-throughput.md)。

## 5. 实现前必须完成的验证

- 状态模型覆盖 alloc、本地 free、并发 remote free、exchange、分批 backlog、slab 退休和线程 detach；证明每个合法 slot 恰好归还一次。
- 验证 C++20 多 producer release sequence、CAS 成功后无 slot/descriptor/annotation 访问、full slab 仍在 owned ring，以及两层游标在小预算下连续推进。
- 用有序标签注入旧 backlog、新批次、持续新 orphan、被 pin 或失败的退休项，检查旧批次不被插队、FIFO失败重排不阻断其他责任；将不满足公平性前提的场景明确标记。
- 定向调度暂停生产者于读 head 后、写 next 后、CAS 前、发布后；暂停 owner 于 exchange 后、live 归零前和撤索引时。
- 验证最后一次远程 free 与 orphan/adopt/descriptor 回收竞争，以及 context 地址重用但 token 不复用。
- 在 ARM 弱内存序、32 位原子宽度和高线程 churn 下运行压力测试；使用支持自定义分配器的 ASan/TSan 集成，并记录其盲区。
- 注入 broker metadata 不足、OS commit/map/protect 失败和回滚失败；已有对象仍可合法释放，Pending 不泄露为可分配空间。
- 多 arena 并行测试隔离、错误 arena handle 的诊断合同、独立销毁及无全局隐式路由。
- recoverable 模式独立验证 durable free 的提交点；普通 remote enqueue 不能使它提前报告持久成功。

当前验证均为待执行项目；详细实验方法见 [基准与验收](benchmarking.md)。
