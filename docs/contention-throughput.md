# 高并发、积压与吞吐边界

本设计说明普通路径为什么能避免多数共享竞争，以及哪些压力必须显式处理。所有容量、批量和公平性规则是待实现的设计；没有运行结果，不能据此声称领先任何 allocator。数据布局以[字段规格](metadata-layout.md)为准，发布与回收证明以[并发协议](concurrency.md)为准，测量方法见[基准规范](benchmarking.md)。

## 1. 先区分四种保证

| 术语 | 本设计可以承诺的内容 | 不能推导的内容 |
| --- | --- | --- |
| 安全性 | 同一 slot 不会被合法调用同时发给两个用户；范围/描述符在仍被引用时不复用 | 暂停线程必然恢复、操作必在期限内完成 |
| 原子 lock-free | 快路径所用原生宽原子经目标验证，不回退到隐藏互斥锁 | 整个普通 API 无锁、每个 CAS 生产者不饥饿 |
| 工作量预算 | drain、class 检查、维护和RT扫描有明确的计数与停止位置 | 等锁、缺页、OS、I/O或线程调度的墙钟上界 |
| 条件性最终进展 | 稳定有限集合、持续调度、充足维护预算、资源可得且无冻结时，已接收归还/已领取责任最终推进 | 休眠 owner 自动被接管、永久坏页被成功回收、过载下仍保持固定延迟 |

普通 malloc/free 可能进入互斥锁、broker 和平台慢路径；整个库不标成 lock-free。MPSC push 不等待 owner 运行，但 CAS 竞争可使个别 producer 长期重试。RT 的固定步骤与普通模式的高平均吞吐分别验收，不把一个模式的性质移植给另一个模式。

## 2. 各路径的共享成本

| 路径 | 常态访问 | 真实竞争点 | 当前控制措施 |
| --- | --- | --- | --- |
| 私有小对象 alloc/local free | 当前 ctx、class、owner slab；定位时读稳定索引 | cache-line迁移、实际缺页；控制面访问必须协调 | owner-only本地字段；不做arena全局统计/quota RMW |
| remote free | 稳定索引、owner身份、自己的next、目标slab remote head | 同一head上的CAS与cache coherence | 每slab独立入口，发布后不再访问归还内存 |
| owner接收remote | acquire exchange与私有backlog | exchange与producer CAS相互干扰 | backlog未空不exchange；普通慢路径总D；显式维护三维预算 |
| 无ctx小对象 | 预建shared heap | 一个arena一个shared heap锁 | 合同明确串行；高吞吐接入显式ctx |
| 新slab/大对象/page请求 | quota、metadata记录、broker、OS | 单broker锁、metadata pool、growth gate、commit及映射 | 额度按span转移、预检预留、Pending、OS在锁外 |
| orphan/retirement | 中央责任队列、目标heap、broker | central锁与后续独立锁 | 独占claim，禁止嵌套central/heap/broker；失败排队尾 |
| durable操作 | 稳定范围pin、提交准入、WAL/checkpoint和flush | 串行提交、持久屏障、checkpoint和设备 | 独立报告；I/O锁外不代表提交可任意并行 |

“没有全局快路径原子”特指普通私有小对象数据路径不逐对象更新全局统计/额度/回收epoch。必要的 per-slab owner/remote 原子读取或RMW仍存在，显式进入控制面、无ctx和持久模式也有自己的同步成本。不能以统计方便为由悄悄增加每次alloc/free共写的arena计数。

SlabDesc三个64B区域分开owner与remote写入，减少常见缓存线争用；真实cache-line大小、布局、对齐和各架构上的收益仍要测。不能把64B分区当作所有平台上零伪共享的证明。

## 3. 从逻辑free到可再次分配

普通归还至少有四个不同事件：用户放弃对象、remote CAS成功、owner接收并计入本地空闲、完整span归还broker。free返回快不等于内存已经能服务另一个heap。

设在同一观察窗口内，`P`为成功发布节点数，`R`为owner已接收节点数，`Q`为remote head和backlog中节点总数。对于无初末悬空统计的稳定快照，有 `Q_end = Q_start + P - R`。仍在CAS循环中、尚未成功发布的调用不在Q里，但继续计入slab live保护；slab live还包括用户持有的对象和Q。测量工具必须说明它统计的是哪一阶段。

`λ`表示工作请求到达率，`μ_publish`表示发布服务率，`μ_drain`表示owner接收服务率，`μ_retire`表示span/后端回收服务率。它们对应不同单位和瓶颈：节点/秒不能直接与页/秒比较，多个size class也不能只用节点数解释字节积压。即使 `λ_free < μ_publish`，只要长期 `P > R`，Q和不可跨heap复用的容量仍会增长；新分配可能先耗尽私有/metadata/commit额度。

容量有限使队列不可能无限占用新节点，不能因此称过载已解决。intrusive队列复用已分配slot，因此队列无需扩容；代价是已释放容量仍可能滞留。应用侧等待队列不是allocator的回退空间，测试与产品都不能靠无限排队掩盖容量失败。

### 批次与轮转

- remote批次按栈序，owner保留当前有限backlog直到处理完；新发布不能插到旧backlog前。
- 每heap维护使用44类cursor和每类全部owned slab环。full slab也在环内，否则它的remote归还可能永远无人发现。
- 普通分配通常路径检查active与一个同类候选；central/refill失败后，在既定一次回收/重搜中检查当前heap相关class整环的已本地可用slot，避免遗漏已有空位而错误报告容量不足。该有穷扫描持owner lease/shared锁且不改变环成员；全部drain合计仍不超过D。其他冷class的维护来自显式flush及应用策略。
- 显式flush沿游标继续，objects/steps用完即停；不能每次从class0重扫，也不能预算用完后把游标转一整圈又回起点。
- arena maintain每调用另在shared锁下沿其既有双游标访问至多一个class/slab并处理至多D个对象，覆盖已adopt但没有新分配的冷类；不获取活私有heap的所有权。
- central和retire采用循环FIFO，已有record提供next；一次maintain每类最多领取一个8条batch到固定栈数组，预算不足时少取。失败项batch末移到队尾，本调用不再领取；不依赖可能被其他maintainer回收的初始tail哨兵。claim后才能离队执行，不能同时属于old heap与central。终态隔离RangeTxn退出active Pending后也进入retire队列，以既有active_next互斥复用，不需要无游标遍历全arena寻找失败事务。

若维护集合稳定、预算持续为正且能容下一个完整工作项，轮转可避免“热点永远把冷项挤到后面”。这不是任意动态流量、任意锁调度下的单调用成功保证；arena maintain先给orphan首项、retire首项和shared一次访问保留预算，再处理batch余项。持续足够三类完整工作的预算才支持三类间进展，详见并发协议。

## 4. 休眠、退出和重新使用

| 场景 | 保留的安全条件 | 如何继续 |
| --- | --- | --- |
| producer在CAS前暂停 | live未扣，自己的slot/descriptor仍保留 | producer恢复后重试；owner不能退休该slot所在slab |
| producer在CAS成功后暂停 | 此后不再访问slot/descriptor；owner可接收甚至复用 | producer只完成自己的ctx/栈收尾 |
| owner在exchange与backlog落位之间暂停 | 链受执行中owner lease保护 | 等其恢复；不能按“head为空”强抢 |
| owner持有heap但长期休眠 | 私有状态不被其他线程改写；remote入口稳定 | 应用调度owner flush/safepoint/detach，或等待下一次调用 |
| 正常detach且仍有live对象 | old owner最后写先于central发布；原descriptor和入口保留 | adopter唯一claim，在目标heap中完成挂接后发布owner身份 |
| 线程被异步杀死 | 不提供任意指令处恢复owner/lock的保证 | 由应用进程级故障策略处理；不能伪装成安全detach |
| 控制面查询持pin | 已退休记录与查询需要的存储不得复用 | pin释放后按退休协议清理；不能偷用合法free的live证明 |

arena身份、owner token和对象/映射generation按各自契约使用，不能相互替代。owner token不回绕；context地址重用不恢复旧身份。裸指针在地址重用后的用户ABA仍属于调用者生命周期责任，诊断不能赋予陈旧指针权限。

清理所需队列节点/归属保存在已存在的记录内。metadata短缺可以推迟物理清理，但不能使合法VOLATILE free失败或让逻辑已归还对象重新变成live。RECOVERABLE_METADATA以提交确认与health门禁为先：提交未确认不能改payload或复用，WRITE_FAULTED和结果未知的处理见[持久协议](persistence.md)，不套用普通intrusive free捷径。

终态隔离X保留steps、cleanup_list和孤立backend的恢复证据，maintenance从既有游标推进；不能只记一个QUARANTINED标签就释放证据。它与Slab/Large/Mapping/RT退休记录对同一OS资源的清理责任互斥，重试失败只重排现有责任，不生成第二个可执行owner。

## 5. 配额、缓存与metadata争用

额度按slab/context/范围事务取得与交还，不按每个slot更新。默认私有span上限为每ctx `4S`、全arena `C/4`；空span保留上限为每ctx `S`、全arena `C/16`，还受其他额度共同约束。参数权威为[配置](configuration.md)，不能把这些默认值误写成用户永远拥有的保底容量。

私有额度耗尽不允许偷偷扩成无限线程缓存。先在本线程既有候选和一次有限回收中寻找可用slot；需要从central取得slab时必须转移相应额度；仍无法取得私有span时可按既定shared fallback策略进入shared heap，容量和commit仍归所属arena。shared竞争是可见代价，不能把它记成私有fast path成绩。任何回退均不使用系统heap或第二个reservation。

空span退出private保留后仍可能处于retired、Pending、后台清理失败或quarantined状态。只有broker最终确认可分配后才记作Free；quota、地址所有权与committed charge分别按事务点转移，不把“空”“已经逻辑free”和“可供任何heap分配”合成一个统计值。

metadata pool锁只在记录申请/归还等慢路径访问；broker仅消费提前取得的records。普通元数据不足时释放当前预检持有的临时quota/记录，再独立进入growth gate，避免多个申请相互占着最后Pending记录等待扩容。一次成功growth发布后调用者重新预检；不得在broker或heap锁内触发growth，也不得递归调用普通malloc。

growth gate是明确的串行点；大量线程同时首次建立ctx、slab或稀疏PageTile时可以形成惊群和延迟峰值。基准必须覆盖冷启动、跨tile突发、16MiB小arena和context churn，不能只用预热后零metadata增长的曲线代表全部负载。可由显式准备路径提前登记、预提交/预触和建立工作集，代价计入启动耗时、metadata和commit。

## 6. 压力下的有限退路

| 条件 | 允许的下一步 | 到达边界后的行为 |
| --- | --- | --- |
| 当前候选无本地slot | 用剩余D处理active/一个候选的归还，尝试central/refill | 失败前一次重搜相关owned整环中的本地可用slot；仍不足按真实可复用容量和额度返回错误 |
| 私有quota不足 | 安全点移交可移交span，或使用同arena shared路径 | 不能超额保留或从外部heap补足 |
| 只有休眠owner中有空闲 | 请求应用在既有safepoint调用flush/detach | 不等待所有owner、不强抢；本次请求可以失败 |
| 预检计划变化 | 释放临时预留并至多一次重新规划 | 第二次失配在OS前返回BUSY；独立metadata增长可已完成 |
| ordinary metadata不足 | 依增长协议尝试一次所需增长/重检；free转交已有退休记录 | 新操作在无用户范围副作用时返回METADATA_LIMIT；清理责任不丢失 |
| OS提交/映射失败 | 按effect、已预留rollback资源及原内容保持证据判定；补偿每步至多一次 | 能证明原合同恢复才报回滚；内容已丢失或身份/效果未知则保留责任并报告实际片段及隔离 |
| 维护工作大于剩余budget | 保留工作，推进其他能完整执行的有预算项 | 不暗中突破pages/steps；返回真实工作量与余留提示 |
| 持久write/flush失败或结果未知 | 按已知/未知效果与持久状态机处理；未知主文件结果锁存health，已确认提交仍须完成内存发布 | 已知无效果错误不一概WRITE_FAULTED；checkpoint补镜像等保留其准入门禁，旧指针不重复free、不提前复用 |

每次普通申请至多一轮配置允许的回收和一次重搜；最后相关class整环搜索是有穷的，但不是固定小步数。它不排除锁、CAS或OS等待，也不构成RT承诺。预检重规划次数与内存压力回收轮数是两个独立计数，不能嵌套成无限“再试一次”。失败事实、checkpoint门禁与可执行退路见[失败与恢复](failure-recovery.md)。

## 7. 批量的收益与代价

增大D可摊薄exchange和函数开销、加快归还字节复用，同时会让执行drain的调用承担更多工作并延长shared锁占用；减小D可降低单次工作量，但可能提高Q、span需求和失败率。因此候选必须同时报告吞吐、调用p99/p99.9、publish到reuse延迟、积压字节、commit/private峰值以及maintenance CPU。

不能靠只测free发布耗时宣布跨线程归还已经解决；也不能把owner接收的CPU藏进未计时维护线程。比较D时固定总 `max_objects/max_pages/max_steps`、线程拓扑、工作集和维护频率；只有明确测“总服务预算变化”的实验才改变这些量。

首版不加入基于无字段压力epoch的自适应控制、perCPU cache、remote路由目录或后台线程。任何新优化需先说明所需字段、16MiB/32位成本、暂停与移交协议和失败退路，再由证据决定是否修改默认策略。

## 8. 优化前的证据门槛

1. 用私有、many-to-one remote、shared无ctx、broker重负载、metadata增长和持久I/O分别定位串行点，不能用“线程越多越慢”直接推断需broker分片。
2. 同时运行闭环饱和实验与开环到达率实验。记录offered/admitted/completed/rejected、生成器遗漏、队列上限和积压斜率；达到某吞吐但积压持续增加不算稳定服务能力。
3. 使用受控暂停与不均匀线程工作量，检查每线程最小完成率、最长无进展窗口和维护服务分配。平均吞吐不能掩盖某个producer或owner长期饥饿。
4. broker分片、shared分片、batch调整或缓存扩张只有在其真实瓶颈被测出、可复现收益超过噪声，且尾延迟、资源上限、碰撞和恢复合同仍满足时进入ADR。

计划中的验证包括发布后零访问、FIFO和双游标的小预算调度、暂停owner与过载服务率、开环测量完整性。具体验证ID由[验证矩阵](validation.md)统一管理；本文不维护第二套完成勾选。
