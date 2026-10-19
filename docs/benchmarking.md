# 基准、性能目标与验收

“领先业界的低延迟高吞吐”是本项目要用证据追求的目标，当前未实现、未运行基准、没有排名结论。首版先冻结正确性、资源和接口合同，再通过可复现实验寻找吞吐、尾延迟、碎片和可控性之间的优势。

## 1. 比较原则与证据等级

### 能力对等

比较系统 allocator，以及在相应平台可部署的 mimalloc、jemalloc、TCMalloc、snmalloc、rpmalloc 等。记录真实版本和支持能力，不要求所有库在所有平台出现，也不把构建失败当作 SuperAlloc 的胜出。

- 通用 malloc 基准比较等价 malloc/free/realloc/aligned 语义和线程模式。SuperAlloc 使用独立函数接口，测试框架通过明确 adapter 调用；产品不进行进程级替换。
- 单 arena、固定容量、外部 backing、页映射、recoverable 和 RT 是独立能力维度。竞争对手不具备某项能力时标记“不适用”，不能给其记零吞吐后汇总胜率。
- 配置一致的 alignment、预提交/预触、清零策略、内存上限和线程绑定；同时保留各库默认配置与合理调优配置，公开差异。
- 持久日志、强制清零、debug poison、guard、统计快照和 sanitizer 可能改变开销，应逐项说明是否启用；不能只为对手打开额外功能。
- 不把普通 VOLATILE fast path 与同步 durable 操作混在同一排行榜。RT 也需同时比较步骤预算、容量成本、成功率和时间分布。

结果分为微基准、代表性合成负载、真实应用集成三层。微基准只能解释机制，不能单独证明数据库、游戏或语言 runtime 的端到端优势。

### 可发布结论

只有预先公开的 workload/机器矩阵、适用基线、统计方法和原始数据共同支持时，才能描述具体范围的领先。不得用单一对象大小、仅平均吞吐或最有利一次运行推导“全面领先”。

## 2. 环境与测量方法

每次结果保存以下可机器读取的 manifest：

- 操作系统、内核/版本、Android API level、设备型号、CPU 微架构、核数、SMT、NUMA、缓存、内存大小和 OS page size。
- 架构及位宽：至少覆盖 x86-64、ARM64，另列 32 位 Android；32 位容量上限与可保留连续地址范围独立记录。
- 编译器/版本、C++20 配置、优化与链接选项、调试及安全功能、allocator 版本/提交、构建步骤。
- arena 容量、backing 类型、所有预算、slab/size-class 表、远程 drain 预算、RT pool 布局/scan 预算、锁页和大页设置。
- CPU affinity、NUMA 绑定、频率策略、温度/降频情况、后台任务、容器或虚拟机、测量时钟及其分辨率。
- workload 配置、随机 seed、对象生命周期/跨线程分发规则、预热和测量时间、重复次数、计时采样方式。

手机需单独记录供电模式、温控和应用前后台状态；不把降频后结果与冷机结果直接并列比较。异构 CPU 必须标记核心类型和线程映射。

### 重复、采样与统计

1. 固定 workload 后先做预热，直到配置的稳定判据满足；冷启动是另一项实验，不能被预热抹去。
2. 每个候选/场景至少做 10 次独立进程重复，随机或交错运行顺序；记录无效运行及剔除理由，不能事后只保留最好结果。
3. 默认每次稳定测量不少于 10 秒；高代价平台可以使用明确记录的替代配置，不与不同测量条件静默合并。
4. 吞吐以 run 为统计单位报告分布和置信区间。延迟报告 p50、p95、p99、p99.9、最大观测值及样本数；最大值不是 WCET。
5. 发布 p99.9 至少获得足够样本：初始要求每场景累计至少一百万个有效延迟样本，并保留各 run 的分位数和直方图。样本不足时不发布该分位数结论。
6. 微秒以下操作的计时开销需单独校准。吞吐批量计时和逐操作延迟采样分开运行，公开采样比例与扰动；不能从一批平均耗时伪造尾延迟。
7. 开环到达率实验记录排队时间和服务时间，避免 coordinated omission；闭环饱和吞吐实验明确其含义，不能把两者混为 SLA 结果。

任何性能阈值的判断均要求超出既定噪声界，并附带置信度和内存成本变化。

### 闭环与开环分开回答问题

闭环实验固定每个worker的对象工作集、最大未完成请求数及生命周期分布，完成一个操作后才发起后续操作。分别测单线程热路径、各线程私有ctx、many-to-one remote、无ctx shared和broker重负载；worker数量、迭代依赖与维护执行方必须公开。它回答给定并发和工作集下的饱和吞吐，不能模拟外部请求持续到达时的排队SLA。

开环实验的请求计划由独立时间轴产生，不因allocator慢下来而降低下一请求的计划到达率。每项请求记录 `t_scheduled`、`t_dispatch`、`t_complete`，分别给出计划到完成、等待派发和实际执行延迟。固定间隔与固定seed的随机到达各自成组，不在结果好坏之后改变分布。生成器、队列与时间戳缓冲预分配在被测arena之外，费用单列，避免把框架自己的分配计入allocator成绩或无意耗尽其容量。

每次开环运行同时报告 offered、admitted、completed、rejected、测试队列drop、生成器missed deadline和未完成数量；到达计划跟不上时不能把漏发请求从分母删除。请求队列有预先声明的容量和满队列策略，过载实验有有限运行时长，不无限catch-up或无限使用框架内存。生成器资源不足的运行标记无效或容量受限，不当作被测库容量上限。

在先导实验测得的服务能力附近扫描到达率，报告测量窗口内吞吐、积压量及其斜率。稳定服务能力要求预先定义的窗口内积压没有持续正增长，同时满足尾延迟、拒绝率和资源门槛；不能把短时间完成率高于长期服务率当作可持续吞吐。停止新到达后继续测排空时间、残留资源和maintenance CPU；到截止时仍未完成的请求属于删失样本并计数，不能悄悄排除后只发布完成者p99。

开环采集的计划到完成时间用于避免 coordinated omission。已经漏掉计划请求的闭环直方图不能靠事后插补成为同等证据；任何校正方法都需保留原始结果、假设及局限。

### 吞吐、公平性与积压是一组指标

记录每个线程/owner的完成率和延迟分布、最小线程吞吐、最长无进展窗口、等待锁分布，以及有仪器时CAS重试分布。异构工作量的线程分别按offered负载归一化，不能用原始吞吐差异直接判断饥饿。可附Jain公平指数，但必须给出输入 `x_i` 和公式 `(sum x_i)^2 / (n * sum x_i^2)`，不能用单一分数取代线程明细。

remote实验至少区分free发布、owner接收、slot被再分配和完整span归还broker的时点与字节数。many-to-one free返回很快但owner backlog持续增长时，不能报告为稳定高吞吐回收。给出remote/backlog峰值、publish到reuse分布、owner私有滞留、retire/quarantine责任数和排空时间；不把逻辑已free容量算成broker已可用容量。

正式吞吐构建不得为每次操作新增全局测量原子。使用调用者外部工作集记录、线程私有采样和owner协作快照；精确全局queue计数需停顿/协调，其开销单独标记。CAS重试、锁等待和逐节点时点可在独立仪器构建中采集并测量扰动，不能把该构建的吞吐与未插桩对手直接混比。

## 3. 场景矩阵

| 维度 | 必测场景 |
| --- | --- |
| 对象大小 | 每个 size class 边界前/等于/后；16 B 到大 extent；真实应用尺寸分布 |
| 对齐 | 默认 16 B、cache-line、系统页及显式更大对齐 |
| 生命周期 | 即分即放、批量生存、随机寿命、长短寿命混合、长寿命对象夹杂短对象 |
| 线程 | 1、物理核心数、SMT 全线程、超额线程；创建/退出/重新登记 churn |
| 释放来源 | remote 比例 0%、10%、50%、100%；单生产者/多生产者/多 owner；producer-consumer |
| 容量 | 16 MiB、小容量压力、中等容量、高占用及 64 位 4 TiB 保留地址测试 |
| 碎片 | 不同 size class 交错、部分空 slab、稀疏长寿命对象、aligned gap、固定映射洞 |
| 混合页面 | malloc 与 page allocation 交错；partial unmap、fixed-no-replace、失败回滚和页范围竞争 |
| 线程状态 | 活跃 owner、休眠 owner、退出 orphan、协作 trim 和内存压力 |
| 共享竞争 | 全部无ctx的单shared heap；混合ctx/无ctx；broker页操作热点；metadata首次增长与跨tile突发 |
| 维护公平性 | full slab仅remote归还；单类热点与44类稀疏；小预算跨调用游标；新orphan持续追加与失败退休项 |
| 过载与恢复 | 开环到达率跨越实测服务能力；有限队列；停止到达后排空；owner暂停和恢复；quota/metadata/commit分别耗尽 |
| backing | 热匿名、首次缺页、预提交预触、文件热页、文件冷页/IO；分开报告 |
| 多 arena | 独立 arena 并发、同线程多个 context、单 arena 热点与多个 arena 分散负载 |
| recoverable | 单操作 durable alloc/free、日志/checkpoint 压力、显式 persist、恢复扫描时间 |
| RT | K = 1/64/N、空扫描、集中归还、伪共享、待回收导致 WOULD_BLOCK、固定配额耗尽 |

4 TiB 只在支持的 64 位地址空间测试；“成功 reserve 4 TiB”不等于“实际 commit 或触达 4 TiB”。实机 RAM/backing 不足时应把最大有效 commit 实验与虚拟范围实验分别列出。32 位 Android 按其独立上限和实际可保留连续范围验收。

### 暂停、批量与调度实验

受控暂停producer于读head后、写next后、CAS前及成功发布后；暂停owner于exchange后/backlog落位前、最后live扣减前和移交前。先用模型验证安全性，再测长暂停与恢复；实现不得为让实验继续而强抢活owner。结果分别给出未暂停线程进展、被保留容量、请求失败与恢复排空时间。

比较D时使用若干预先声明的正整数，保留默认64，并固定显式维护总objects/pages/steps和频率；不能一边增大批量一边增加总服务预算却只归因于batch效率。报告慢路径、drain调用和普通热路径各自延迟，避免大量极短热调用掩盖少量极慢维护。RT扫描K单独试验，不借普通D改变RT步骤合同。

线程数量覆盖低于、等于和超过可用硬件线程；加入少数慢producer、单个慢owner和不均匀对象尺寸，检查吞吐随线程数的饱和点及jitter。报告按固定窗口统计的吞吐/延迟波动，窗口长度进入manifest。可用平台记录锁占用、coherence、抢占及page fault；不可用的硬件计数标注不可用，不填零。

### 代表性应用

- **语言 runtime**：对象尺寸谱、跨线程任务传递、短命对象与长期 runtime metadata；明确 GC/引用计数框架如何调用 allocator。
- **游戏引擎**：帧内突发、任务线程池、资源加载/卸载、长寿命资源与临时对象；观测整帧 p99/p99.9 和内存峰值。
- **数据库**：并发请求、缓存淘汰、页块与小对象混用、长事务/快照保留；持久化实验区分 allocator metadata 与应用 payload 事务。

应用实验必须报告完整输入和端到端指标，不能只报告 allocator 子函数速度而忽略 CPU、I/O 或内存峰值回退。

## 4. 指标与验收门槛

### 正确性先行

性能候选进入比较前必须通过：

- allocator 状态模型、尺寸/对齐/范围溢出测试以及随机 alloc/free/page-map 操作序列。
- MPSC 发布、live 晚扣、backlog 分批、最后一个 remote free、orphan 接管和 descriptor 生命周期竞争测试。
- 多producer release sequence、CAS成功后的零归还内存访问、owned ring覆盖full slab、双层维护游标以及central/retire FIFO的失败重排。
- RT release/acquire、固定扫描预算、跨线程必成功 free、返回 WOULD_BLOCK 和 quiescent teardown 测试。
- ARM 弱内存序与 32 位原子构建/运行检查；支持的 sanitizer 检查及自定义 allocator annotation。
- OS/metadata 故障注入、Pending 回滚或隔离、最小 arena 自举、多 arena 隔离和资源预算检查。
- recoverable 模式每个日志提交/写入边界的崩溃注入与恢复不变量检查；flush 错误不能提前报告 durable success。

不允许以吞吐优先为由跳过任何所有权、发布或生命周期不变量。

### 同时报告的性能和资源指标

吞吐、每操作 CPU cycles、延迟分布、CPU 占用、cache miss/coherence（平台可测时）、remote backlog、broker/shared/metadata/central 等待、OS 调用/缺页数量，以及 live/requested/usable/private/metadata/committed/reserved/resident 各项内存。另列维护与后台测试辅助线程的CPU，不把成本移出计时区间当作吞吐提升。

碎片至少同时报告 slot 内浪费、partial slab 容量、空闲总量、最大连续空闲范围和 extent 数。reserved VA 不作为实际物理内存消耗，resident 采样不作为完全精确的 allocator 计数。

普通小对象默认不跟踪逐slot requested size。基准框架可以依据自身workload记录计算requested live/slot浪费，但应区分框架外部测量与allocator内建统计；不能把不可用字段填0，或只对SuperAlloc开启有额外侧表成本的精确追踪而不说明。相关空间成本见[自举与容量预算](metadata-budget.md)。

RT 额外报告单次最多访问/回收 slot 数、完整扫描维护轮数、WOULD_BLOCK 比例及每 slot 元数据成本；一味减小 K 提升每次调用速度但显著增加失败率，不算免费改进。

### 初始门槛

第一份通过正确性并完成环境冻结的结果作为内部基线；此前不设虚假的绝对 ns 或 ops/s 保证。之后在固定矩阵内：

- 吞吐下降超过 5%；
- p99 延迟上升超过 10%；
- 峰值 committed bytes 上升超过 5%；

若任一变化超出已测噪声范围，默认阻止合入性能敏感改动，直到解释原因、给出对其他关键指标的收益，并由 ADR 明确接受取舍。p99.9、OOM 成功率、预算违反或正确性退化即使未触达这些百分比，也需要单独审查。

这些是工程回归门槛，不是行业领先的证明。对外优势需要对每个适用竞争对手提供同等条件下、具有统计支持的具体结果，不能用总体平均掩盖关键场景退化。

## 5. 执行顺序与产物

1. 建立状态模型、故障注入及测试 adapter，固定场景名称、结果 schema 和环境 manifest。
2. 实现可运行的最小正确 baseline，完成多平台与 32 位原子检查。
3. 收集首轮结果，确定瓶颈是 size class、slab 周转、remote CAS、broker、缺页还是持久 I/O。
4. 每轮只改变可解释的机制，保存前后配置和原始数据，再按同一矩阵复验。
5. 达到通用模式正确性和资源门槛后，再评估 NUMA、broker 分片、专用 RT profile 等复杂优化。

每轮产物包括：原始样本或直方图、运行日志、环境/配置 manifest、候选版本、失败/剔除记录、统计脚本版本，以及一页说明适用范围和代价的结论。性能图必须可从归档数据再生成。

开环产物还包括到达计划/seed、队列容量与满队列策略、offered/admitted/completed/rejected/drop/censored计数、生成器deadline记录、积压时间序列和排空区间。公平性与batch实验保留每线程明细、维护预算、D/K、受控暂停位置和时长。架构优化的依据对应[竞争与吞吐边界](contention-throughput.md)中的具体瓶颈，不能仅保留一张总体吞吐图。

当前仓库只建立设计和实施待办；不得填入未经执行的基准数字、平台通过标记或性能排名。
