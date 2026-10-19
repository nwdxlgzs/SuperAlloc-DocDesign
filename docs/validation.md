# 验证策略与验收矩阵

状态：测试设计，尚未执行分配器测试。当前仓库只有文档；以下“通过条件”不代表已通过。

## 1. 验证分层

| 层次 | 方法 | 能证明什么 |
| --- | --- | --- |
| 文档与模型 | 接口场景表、状态转移、预算算式、锁与内存序审查 | 设计内部一致、契约可实现性与待验证点可追踪 |
| 纯逻辑模型 | 参考区间集合、确定性随机序列、小状态穷举 | 分割合并、碰撞、错误结果、回滚的所有权不变量 |
| 可控 VM 后端 | 每一步 map/commit/protect/flush 注入失败 | Pending、隔离和持久结果未知处理 |
| 实机集成 | 四系统、真实页大小、32/64 位、文件与线程 | OS 语义、原子实现和平台资源限制 |
| 并发与工具 | 调度扰动、弱内存序模型、TSAN/ASan 等适用工具 | 发布/回收竞争、越界和生命周期错误；工具通过不是完整证明 |
| 持久故障 | 进程终止、日志撕裂/乱序模型、受控断电 | 分别验证进程恢复、持久协议和特定存储链路屏障 |
| 性能 | 固定配置、原始样本与竞争对手适配 | 特定场景的延迟/吞吐/空间成本，不推断全面领先 |

参考模型可以在测试进程使用外部容器；被测核心不能因此放宽区域内自举。测试工具、构建和结果文件不计入 allocator arena，但必须与分配器自身的内存统计区分。

## 2. 场景矩阵

| ID | 场景与操作 | 通过条件 | 对应任务 |
| --- | --- | --- | --- |
| V-01 | 16MiB 最小 VOLATILE 创建、分配至额度、释放、销毁 | 管理开销目标单独核算；无外部堆；free 在 metadata 耗尽时仍成立 | I-04、I-05 |
| V-02 | 支持设备保留 4TiB，只触达少量跨度很大的页 | 不建立全域密集目录；创建成本与实际元数据有关；reserve 与 commit 分开报告 | I-01、I-04 |
| V-03 | Android32 容量、offset/length 加法、round-up、calloc 乘法边界 | 越界在修改前拒绝；不依赖 lock-free u64；从不截断成功 | I-03、I-05 |
| V-04 | 所有 size class 边界、0 大小、扩展对齐、realloc 成败 | 16B 保证、数据保持、失败不丢旧块、零语义符合 API | I-05 |
| V-05 | 双实例同线程/多线程并发、一个耗尽或后端失败 | 无锁/额度/描述符串扰；另一实例可继续工作 | I-05、I-07 |
| V-06 | malloc 与 page map 同时从两侧增长、中间空洞和固定位置请求 | 地址唯一所有权；不覆盖 metadata、slab、RT 或 Pending | I-04、I-06 |
| V-07 | 总 free 充足但无足够连续范围；不同 view 边界相邻 | NO_CONTIGUOUS_RANGE 或受控整理；不谎称总 free 可立即分配 | I-04、I-09 |
| V-08 | 每个节点预留、metadata grow、split、publish 前后失败 | 无递归自举、无丢失所有权；应急额度不被普通请求耗尽 | I-04 |
| V-09 | 每个 backend 子操作及其回滚失败 | 完整恢复或 PARTIAL_FAILURE/Quarantined；未知范围从不复用 | I-02、I-06、I-09 |
| V-10 | 中间页 unmap，洞映射新文件，随后使用旧 handle | 两侧数据与访问连续；旧 handle 不操纵洞中新 owner | I-09 |
| V-11 | ordinary/splittable 配额边界、估算与执行之间竞争 | 超额预检失败或并发重检失败；不静默降级；实际资源可查询 | I-02、I-09 |
| V-12 | 文件 offset、PRIVATE COW、SHARED persist、保护边界、resize | 页对齐检查正确；私有修改不宣称写回；unsupported 无副作用 | I-02、I-09 |
| V-13 | 0/10/50/100% remote frees，producer 在 CAS 前暂停 | 未发布/未接收 free 仍保护 descriptor；无提前回收和丢对象 | I-07 |
| V-14 | 最后对象 remote free、owner drain、backlog、detach/orphan 接管 | 发布后不再访问旧 descriptor；唯一 owner；退出不丢 live 对象 | I-07 |
| V-15 | owner token/ctx 地址复用、累计 token 耗尽、动态 radix 建路竞争 | 无身份 ABA；token 不回绕；路径不丢分支；目录回收需 quiescence | I-04、I-07 |
| V-16 | 缓存预算、睡眠 owner、OOM 维护、精确/并发统计 | 无无限等待、无预算外扩张；统计标明精度，不强抢 owner 结构 | I-07、I-13 |
| V-17 | RT 全部对象已分出，owner 暂停，其他登记线程归还 | 所有合法 free 成功；无 queue-full、CAS 重试、OS 或动态分配 | I-08 |
| V-18 | RT K=1/64/N，归还恰在扫描后，slot 多轮复用 | 单调用检查/回收<=K；下一完整轮覆盖该位置，有应用 HB 的完成归还一轮接收；未找到只报 WOULD_BLOCK | I-08 |
| V-19 | RT prepare 任一资源失败、锁页失败、尝试未登记线程 | 不进入部分 ready 状态；要求锁页时无静默降级；非法调用清晰 | I-08 |
| V-20 | borrowed buffer 创建与释放、无 VM 能力的 protect/map 请求 | 不解除调用方内存，不执行未授权 VM 动作；RT 初版拒绝 | I-05、I-06 |
| V-21 | 同一主文件别名、硬链接、重复 handle、跨进程竞争、崩溃后重开 | 单管理者；不用 PID 替代 lease；独立主文件可并存 | I-10 |
| V-22 | CLEAN_REOPEN 带 live 对象 clean close，换基址 open，非正常退出 | 成功 close 后对象/root 正确；dirty clean store 不冒充 recoverable | I-10 |
| V-23 | redo、commit、运行时应用、返回、checkpoint 切换各边界中断 | 只恢复完整已提交事务；结果未知不复用；无重复分配 | I-11 |
| V-24 | free 提交前失败、主文件写回不确定、root 引用保护 | 提交前 payload 不改写；COMMIT_UNKNOWN 后 WRITE_FAULTED；原始状态可恢复判定 | I-11 |
| V-25 | persist→root_set、未发布分配、对象内间接引用、外部 session map | orphan 可枚举不自动删除；root 不指外部 map；重开不恢复 native handle | I-11 |
| V-26 | 坏 checksum、日志中间损坏、尾部撕裂、未知版本、页几何差异 | 合法尾部处理与 CORRUPT 区分；不兼容拒绝且不改写原文件 | I-10、I-11 |
| V-27 | API 输出结构较短/长、未知 flag、C 与 C++ 调用、32/64 布局 | 不越界写输出；版本拒绝明确；无异常跨 ABI、无隐式 errno | I-03、I-14 |
| V-28 | 普通 destroy BUSY、quiescent discard、持久 close 保留 live | 生命周期契约正确；不把并发 use-after-destroy 伪装成受支持调用 | I-05、I-10 |
| V-29 | metadata-layout所有记录在各ABI下的sizeof/alignof/offsetof、原子与native控制槽 | 与模型逐项吻合或先修订预算；不能仅断言总数<64KiB，不能隐藏外堆 | I-03、I-04 |
| V-30 | 超大对象、特殊对齐、同tile不同合法起点、释放与realloc | 大对象索引仅返回页且不随对象长度增长；slab/RT覆盖合法起点；控制面范围查询正确 | I-04、I-05 |
| V-31 | requested侧表开/关、小对象realloc与sanitizer | 关闭时相关精确统计不可用；开启后计入metadata；复制不越可读slot范围 | I-05、I-07 |
| V-32 | 控制岛尾部用尽、一次/连续两次metadata增长失败并隔离 | 每次已有记录描述全部结果；不能补足完整应急组则拒绝增长；正常free逻辑成功 | I-04、I-07 |
| V-33 | profile1三种代表容量、显式0、溢出、quota并发预付、替换峰值和增长互等 | 默认公式一致；used+reserved不超限；增长前归还用户ticket；private达限仍能移交/共享，free不因empty额度失败 | I-04、I-07 |
| V-34 | report最坏3k+2、正式重试变化、Pending期间分页、begin/publish代次与饱和、move后半失败 | OS前缓冲不足无效果；PARTIAL_FAILURE完整报告；Pending时BUSY、revision饱和统一STALE_CURSOR；收口信用不回绕；合法free/清理仍继续 | I-06、I-09 |
| V-35 | reserve后bootstrap失败且清理失败、最后控制岛清理失败、未知native close状态 | 固定caller token接管唯一责任；无外堆；旧arena不再可调用；不盲重试已复用FD | I-01、I-04、I-10 |
| V-36 | 固定字段LE编码、CRC向量、padding、slot ID/generation、FREE tombstones与最大值 | 实际字节与格式候选一致；FREE保留代次；整数不回绕；恶意数量/offset在访问前拒绝 | I-10、I-11 |
| V-37 | 两个commit副本撕裂、尾部无commit、中间缺口后有commit、坏redo | 只接受完整连续提交前缀；有效commit配坏redo为CORRUPT；未确认尾部按明确故障模型处理 | I-11 |
| V-38 | 双super发布/镜像每一步失败，旧WAL和CP重用，CLEAN→DIRTY | 镜像屏障前不覆盖依赖；新写入前双DIRTY确认；不会回退到仍称CLEAN却已被修改的旧链 | I-10、I-11 |
| V-39 | R内RESERVED增长、session maps与恢复预算、open覆盖options | R永不进userFREE；metadata地址只预付一次；物理提交另计；重建不足在发布前拒绝且不覆盖有效store | I-04、I-10、I-11 |
| V-40 | RTBindings阻止detach、空pool逻辑销毁与延后物理清理、owner变更 | bindings不悬垂；合法teardown用已有控制资源退休；改变owner须清空销毁再prepare | I-08 |
| V-41 | 新空file_ref初始化、非空坏文件/有效文件create、lease前预读变化 | 非空不截断；lease后完整重验；输入FD不误关闭；namespace与文件内容保证分离 | I-10A、I-10 |
| V-42 | 六类接入生命周期：普通多实例、borrowed、页映射、RT、clean、recoverable | C调用示例有成功和失败退出；不跨线程共享ctx、不跨对象族free、不省略quiescence/业务persist | I-05、I-08、I-09、I-10、I-11、I-14 |
| V-43 | 版本/defaults/ABI/store feature拒绝、导出、发布能力标签及复现归档 | 候选与正式冻结区分；未知必要特性不忽略；未测平台不报VERIFIED；报告链接真实证据 | I-03、I-12、I-14 |
| V-44 | 多view效果四态、执行失败后补偿再次失败、匿名/COW不可逆内容变化 | 按实际效果发布完整片段；回滚不递归重试；未知费用不退；Pending之外没有双重所有 | I-04、I-06、I-09 |
| V-45 | Pending执行者暂停、诊断pin、并发预检失配、配额结算与最终发布交错 | 无超时抢占或双完成者；同mapping变更串行；结算先于可见owner；Pending信用最后退；pin前不复用 | I-04、I-07、I-09 |
| V-46 | 初始化/最后控制岛清理失败，FD与地址被其他owner复用 | token单向移交；旧arena入口不复活；已完成资源不重复清理；未知身份不盲重试 | I-01、I-04、I-10 |
| V-47 | remote发布后立即回收、page/RT逻辑释放后清理失败、持久commit后并发fault | 发布后无slot/descriptor访问；已成功释放不撤销；确认提交完成publish；未知不复用 | I-07、I-08、I-11 |
| V-48 | 多producer CAS、release sequence、owner exchange与backlog、最后free移交 | 原生宽原子正确；next可见且每节点只接收一次；失败序不依赖旧head解引用；不承诺单producer无饥饿 | I-03、I-07 |
| V-49 | 小预算跨class/full slab维护、FIFO双类别固定batch、终态X退休、多个maintainer | 两级游标持续推进；同调用预算不重置；既有记录独占claim；失败不阻塞其他记录，无裸tail哨兵UAF | I-04、I-07 |
| V-50 | owner暂停、归还积压、私有quota满、metadata增长惊群、重复计划失配 | 不强抢owner；本地可用slot有穷重搜；一次重规划后BUSY；无隐式扩额度；恢复维护后按条件消退积压 | I-07、I-13 |
| V-51 | 闭环与开环到达率、生成器落后、维护CPU、逐线程进展和长期过载 | offered/admitted/completed/rejected与未完成数闭合；记录积压和端到端尾延迟；不以漏发请求或隐藏维护伪造稳定吞吐 | I-13、I-15 |
| V-52 | native无效果/部分生效、匿名/COW回滚与reservation失证 | 完整逐片报告；不盲fixed/unmap陌生地址；确认释放才退账；局部隔离不波及可信范围 | I-02、I-04、I-09 |
| V-53 | data persist故障交错redo、commit许可/确认、quota结算及runtime publish | 确定未提交、OK且health故障、COMMIT_UNKNOWN准确区分；已确认commit必发布；故障后无新复用许可 | I-10、I-11 |
| V-54 | checkpoint/close排空persist pin、gate等待者、WAL满自退出、第二super已知无效果失败 | 无遗漏admission、无自等/lease提前释放；CHECKPOINT_HELD唯一续镜像；未确认不回收WAL/CP | I-10、I-11 |
| V-55 | 未知native cleanup、未知pfree重开身份查询、两个arena并行 | 不重试已复用FD/地址或未知free；控制岛与恢复证据不丢；实例故障互不串扰 | I-04、I-10、I-11 |

## 3. 故障注入方式

为每个可失败步骤设置确定性 operation ID 和序号。对同一输入序列重复运行，依次令第 n 个步骤失败，再检查所有权模型、OS 资源计数、保留地址、文件记录与泄漏。

失败前后的判断必须包括：

1. 返回值与可见效果一致，既不假成功也不假装失败无副作用。
2. 新对象不部分发布；旧对象保持合同或明确报告其状态已无法保证。
3. 未知 backing 只能隔离，未知持久结果禁止复用。
4. 元数据、views、文件引用、日志预留和 Pending 记录有归属，不重复释放。
5. 诊断路径本身不递归申请外部堆、不取不允许的锁、不执行 RT 慢路径。

模型后端需能分别表达“调用失败且无效果”“已产生效果但返回错误”“回滚失败”。只模拟第一种无法验证真实 I/O 和 VM 不确定性。

V-44～V-55的详细暂停点与oracle见[状态机](state-machines.md)、[失败与恢复](failure-recovery.md)和[争用与吞吐](contention-throughput.md)。每个调度断点联合检查地址owner、descriptor可达性、native责任、used/reserved、调用结果和durable事实；单看某个state枚举合法不算通过。可控模型注入销毁准入事件验证其守卫，但公开destroy/close始终要求调用方quiescence，不能把并发销毁违约用例包装成受支持容错。

## 4. 并发与平台覆盖

关键调度点包括：读取 remote head 后、CAS 发布前后、exchange detach 后、live 扣减前后、owner token 移交、目录发布、Pending 登记与 OS 调用之间。对这些点进行可重放暂停与交错，不只依靠长时间随机压力。

ARM64/ARMv7 使用 release/acquire 内存序模型和实机压力；x86 测试通过不能代表弱内存序正确。验证编译后的关键原子路径，确认没有未经预算的 libatomic 锁或外部堆调用。

自定义分配器需要配套的 ASan poison/unpoison 或相应 annotation；TSAN 的支持范围、32 位局限和 backend mock 要如实记录。测试工具不适用的系统必须使用替代检查，不能直接标为通过。

Windows 验证 placeholder 的保留完整性和 view 界限；POSIX 验证释放子区间后直接用受控映射恢复 reservation，避免先释放地址再让第三方抢占。真实页大小测试不能用把常量改成 16KiB 的 mock 代替。

4TiB 保留测试只有设备具备地址空间时执行，其他设备明确记录“不具备能力”；另用纯逻辑模型覆盖 4TiB offset，不把模型结果报告为设备成功。

## 5. 持久验证边界

持久测试保存原始文件副本和故障序列。解析器先校验所有长度与边界，再使用其作为索引或读入长度；坏盘数据不能驱动越界、巨量外部分配或任意地址映射。

必须区分三份报告：

- 进程 kill 后恢复：验证运行中断，不证明设备断电。
- 可控日志 torn/reorder 模型：验证协议容忍特定写入失效，不证明所有真实磁盘。
- 实际存储链路掉电测试：仅对记录了 OS、文件系统、设备和屏障的配置给出结论。

`CLEAN_REOPEN` 的验收不借用 `RECOVERABLE_METADATA` 的恢复代码绕过 dirty 拒绝语义。新文件路径的 namespace durability 与既有文件数据同步单独记录。

## 6. 证据与完成条件

每项实现任务的完成证据包括：版本/变更、平台、配置、用例 ID、结果、失败或跳过原因、原始日志位置。文档阶段只检查链接、任务依赖和契约一致性，不创建虚构测试报告。

回归用例保留触发最小序列与 seed；同一机制修正后优先运行受影响场景，再执行对应阶段门槛。性能规范见 [benchmarking.md](benchmarking.md)，不得用更快的结果抵消错误的内存所有权。

平台试验的导航见[实验顺序](platform-experiments.md)，具体PV步骤见[验证剧本](platform-validation.md)。所有V/PV运行项当前均未执行；[需求与文档审计](completion-audit.md)只证明设计交付范围，不替代这些未来实现门槛。
