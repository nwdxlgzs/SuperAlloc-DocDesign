# S-02：配置、额度与维护策略

状态：defaults profile **1 的设计候选**，没有运行调优数据。本文件规定默认值、单位及额度结算；公共字段与数字键见 [C ABI](c-abi-spec.md)，资源预付流程见 [操作协议](operation-protocols.md)。它不把可配置上限当作已预分配内存或成功分配承诺。

## 1. 配置求值与版本

容量 C 为 arena 的全部地址长度，包含管理空间。P 是实际系统页大小；S=max(64KiB,P) 是普通 slab 大小；G=max(64KiB,P,后端 allocation granularity) 是持久控制区域几何。首版支持已验证的二次幂几何。全部公式先用受检查的 u64 计算，向上/向下取整也检查溢出；最后才窄化至本机地址类型。

创建者明确传 defaults_version=1。缺少某 option 表示采用该版本公式，显式 0 表示禁用相应资源，不等同自动。未知键、重复键、未知 flags、不可表示或矛盾的配置在取得用户范围之前拒绝。能力不足返回 UNSUPPORTED，不暗中减小容量或降低持久级别。查询返回**求值后的配置**，便于重放基准。

首版资源上限在创建/打开时确定，运行中不可更改；改变上限需要 quiescent 关闭/重建符合几何的实例。维护每次调用的工作预算独立传入，可以随应用帧或负载调整。未来如需在线限额变更，必须另外定义已预付信用的排空协议。打开持久文件不能用配置改写其固定 R/J/K/N/root 几何；可接受的运行选项见持久格式。

## 2. 十二项资源额度

每项有 limit、used、reserved、peak 四个 u64。peak 记录 used+reserved 的峰值。LimitBlock 的大小仍为 [S-01](metadata-layout.md) 的448B；12项由 metadata pool 叶子锁在慢路径短暂维护，不增加对象热路径全局原子。

| option键 | 名称 / 单位 | VOLATILE 默认 | 持久模式默认 | 计费边界与耗尽结果 |
| --- | --- | --- | --- | --- |
| 1001 | backend_limit / B | C | C | 可访问或仍保留资源的 backing charge；COMMIT_LIMIT |
| 1002 | runtime_metadata_limit / B | alignG(max(1MiB,ceil(C/32))) | R | 向元数据所有权划入的完整地址容量，包括空库存；持久R整段初始计used，物理提交另计；METADATA_LIMIT |
| 1003 | ctx_limit / 个 | clamp(floor(C/1MiB),4,1024) | 同左 | 显式 live ctx + 正在登记的ctx；共享heap不占ctx名额；LIMIT |
| 1004 | private_span_limit / B | floorS(C/4) | 同左，但持久对象前端不使用普通ctx slab | 独占于私有heap的完整slab；LIMIT可触发共享路径 |
| 1005 | empty_span_limit / B | floorS(C/16) | 同左 | 所有heap缓存的全空slab；与private有交集，不能相加算payload；超限退休 |
| 1006 | fine_view_limit / 个 | 16384 | 16384 | 可拆映射的逻辑backend单元，每P一单元；LIMIT |
| 1007 | ordinary_view_limit / 个 | 4096 | 4096 | 普通映射、slab、大对象、内部元数据及持久控制/数据backing单元；LIMIT |
| 1008 | pending_limit / 个 | 64 | 64 | 正在执行的范围事务；增长事务也占名额；LIMIT |
| 1009 | rt_payload_limit / B | floorP(C/2) | 0 | 固定slot载荷，状态/索引另计metadata；LIMIT |
| 1010 | root_limit / 个 | 0 | 64 | 持久根槽总数，空根也属于固定表；LIMIT |
| 1011 | journal_limit / B | 0 | J | 持久WAL固定地址长度；满时checkpoint，不能自动外扩 |
| 1012 | checkpoint_limit / B | 0 | 2K | 两份checkpoint地址长度**合计**；每份K；不足拒绝配置 |

fine/ordinary 是项目统一的逻辑单元预算，POSIX 即使能合并内核 VMA 也按逻辑单元收费；实际 OS view/VMA 数作为不同统计公开。两项互斥计数，不重复给同一个 BackendUnit 收费。实际内核资源可能更早不足，返回 OS_ERROR；不能声称 limit 就是 OS 支持的数量。已解除地址但仍存活的section charge与资源引用，直到OS确认释放才取消计费。

backend charge 的统一含义是后端声明的资源承诺量：匿名已提交/仍保留的资源、文件有效视图覆盖的页和库掌握的其他保留backing；不是工作集或RSS，也不宣称POSIX overcommit保证不会OOM。共享同一section的views按section已承诺容量计一次；Windows解除一个view若不能撤销section提交，继续记账。另报平台原生commit能力与采样，不把这些量混为一谈。旧、新backing同时存在时计入峰值，即使二者映射同一逻辑地址也不能只付一份额度。

借用buffer的backend charge在创建时为C，由调用方预先建立，之后逻辑释放不降低它；库不额外创建OS views，初始buffer只作为一个受控backend单元统计。借用buffer的RT、日志和roots禁用。metadata、私有缓存等仍在C内计算，不因外部提前映射而免费。

这些是不同维度的上限。metadata、RT和private额度不能相加宣称总额可同时使用；broker要求它们的实际地址所有权互斥，backend再限制实际backing。无论多个上限之和多大，都不能超出C。16MiB普通模式只初始取得64KiB控制岛，1002=1MiB并不预先吃掉1MiB。

持久模式的R是固定、专属的METADATA地址域：创建即1002.used=R，未启用部分处于RESERVED子态，不在用户FREE索引中。R内增长仅把既有RESERVED单元启用，不再次增加metadata地址额度；backend、views、Pending及实际记录成本照常预检。ACTIVE和QUARANTINED也是同一metadata域子态。query的broker_free仅报告可供用户所有权申请的空闲，排除R及格式固定区；metadata_reserved_bytes报告已划属总量，metadata_live_bytes报告实际在用记录/缓冲，物理提交由backend charge表达，不能把R预留误读为R已提交。

## 3. 策略参数

| option键 | 名称 | 默认 | 有效范围及含义 |
| --- | --- | --- | --- |
| 1101 | remote_drain | 64 | 1…2^32−1，每个接收步最多处理节点数；普通CAS仍无重试次数保证 |
| 1102 | rt_scan_default | 64 | 1…2^32−1，prepare未覆盖时的K；每个pool固定实际K |
| 1103 | fine_views_per_map | 4096 | 0…2^32−1，还受arena fine总额度和metadata限制 |
| 1104 | private_span_per_ctx | 4S | S的倍数，0禁用私有slab缓存，仍可通过shared heap分配 |
| 1105 | empty_span_per_ctx | S | S的倍数，不得大于private_span_per_ctx；0表示全空即退休 |
| 1106 | persistent_allocation_limit | N | 持久分配目录槽总数；VOLATILE必须省略或0 |

TRACK_REQUESTED=bit0 付费启用小对象requested侧表；DIAGNOSTIC_CHECKS=bit1启用文档指定的附加校验。二者不改变合法调用的所有权合同，也不保证检测任意UAF。首版无用户回调，错误由调用者查询，避免日志/回调递归进入分配器。诊断表和侧表全计入metadata，不偷偷增加默认额度。

ordinary_view_limit、pending_limit、backend_limit和metadata_limit不能设置得小于所选模式实际自举需求；包括初始单元、两套增长应急记录及持久固定区。计算器用实际后端计划检验，不凭一个跨平台硬编码view数量放行。ctx/private/empty/fine/RT/root允许0，其中持久root=0表示不建立可用根槽，持久对象仍可由应用枚举管理。runtime metadata至少能容纳64KiB控制岛及模式所需重建空间。

实体计数1003、1006、1007、1008、1010、1103、1106不超过UINT32_MAX，字节额度使用u64并经当前地址/几何校验。RT每类slot_count、class_count、participant_count各不超过UINT32_MAX，总slot_count可为经检查的u64和；这不引入64位原子。格式几何v1固定G=64KiB、P为4KiB或16KiB，不满足时持久模式拒绝，普通平台能力另外探测。

## 4. 私有容量转移与释放不失败

私有上限限制heap独占的span容量，不限制线程能够累计创建的活对象数量。达到4S默认值后，owner在安全点把选中的slab完整移交central（可带live对象，remote入口不变），释放private信用，再取得新slab；也可以走加锁shared heap。不能仅因某线程已经分满4个slab便永久拒绝其新请求，同时让其他全局空闲无法使用。

普通slot alloc/free不调整额度。新slab附着、移交或最后一个live归还属于慢路径，在这些边界结算整span信用。全空slab若超过arena或ctx空缓存额度，即摘入已有描述符的retire链；这是已逻辑释放、等待后端回收的资源，不能冒称缓存或broker可用。其backing和metadata仍计费。没有额度或OS回收失败不能撤销已成功的free。

shared heap没有私有额度，但仍受C、backend、metadata、ordinary view及全局empty上限。central不能用“shared”命名绕过空缓存上限；带live的central slab是活所有权，空slab则必须受同一empty/retire规则约束。休眠owner上的零散空slot不可强抢，其整个private span已经付费。

## 5. 资源事务

每个同步慢路径在有界栈上保存Q[12] reservation ticket。持metadata pool锁一次检查所有used+reserved+delta<=limit并预付reserved，随后解锁再取得记录信用。只对**新划入metadata所有权的地址字节**收费；从已存在pool取descriptor或在已预付的持久R内启用单元不再次增加metadata地址用量，PoolFamily的reserved_records专指不能给普通请求使用的应急库存。

记录不够且需要增长时，用户操作尚未Pending：归还其额度/records，以独立增长ticket和应急记录扩一次，补齐应急库存后重算用户计划。不能拿着最后一个Pending或backend额度等待需要同样额度的growth。增长失败仍必须足够描述已隔离资源；不完整的下一组应急信用禁止继续增长。

登记Pending后使用预付资源执行OS操作。最终后端效果已确定时仍保持Pending及全部pin，在不持broker/heap/central锁的情况下于metadata_pool锁内完成reserved→used、已确认释放资源的扣账及多余ticket归还；随后才进入broker做不再可能申请资源或调用OS的最终发布，撤销Pending后交付对象/handle。这样其他线程不会看到可退休的新owner却在used中找不到对应费用。回滚只归还已证明不再使用的信用，无法释放的backing保持used并报告隔离/退休状态。进行中Pending的名额与记录如何随收口移交按[操作协议](operation-protocols.md)执行；估算API不预付，正式执行重新检查。

Pending名额是上述顺序中的独立控制信用：它在最终发布前仍实际占用，因此保持在本调用ticket.reserved，等broker撤销Pending后才单独归还。这个短间隔只保守占用已结束事务的名额，不把可见新资源的backing费用留在reserved；其余描述符/pin按责任移交顺序释放。

失败判定按阶段进行，不承诺所有失败原因都同时扫描：先参数/身份/模式，后明确的配置额度，再检查地址可用总量与连续性，最后OS。OUT_OF_MEMORY表示当前可用总容量不够，NO_CONTIGUOUS_RANGE表示总量够但无法形成符合对齐及后端边界的范围。METADATA_LIMIT、COMMIT_LIMIT和LIMIT携带具体option键、used/reserved/limit；OS_ERROR保留原生错误。全局同时变化时，错误仅描述该次检查观察到的阻塞因素。

## 6. 持久默认几何

定义R=alignG(ceil(C/16))，J=alignG(clamp(ceil(C/128),256KiB,64MiB))，K=alignG(clamp(ceil(C/64),256KiB,64MiB))。两份superblock各G，runtime固定区R，WAL为J，checkpoint两份各K；固定保留合计F=2G+R+J+2K，payload候选空间C−F。R只是保留地址上限，匿名runtime按需建立，不能在4TiB创建时全部触页。

默认根槽数H=64。目录上限：

```text
Ncp = floor((K − 4096 − 64H) / 64)
Nrt = floor((R − 128KiB) / 768)
N   = floor(min(Ncp, Nrt) / 1024) * 1024
```

计算前验证减法非负。自动N<1024拒绝该profile；显式N可从1起，但必须满足目录和运行空间校验。768B是默认选型信用，**不保证任意稀疏拓扑都能容纳N个对象**；实际alloc和open依S-01逐项预估descriptor、radix、span、journal scratch与索引，资源不足明确失败。目录槽保存free generation，因此释放并不缩短checkpoint目录；generation耗尽的槽不能绕回复用。

显式1012必须为2G的倍数并拆为等长K；J/R均需G对齐；C为G整数倍，F<C且至少留一slab payload；H/N和固定格式记录必须容纳在K；运行工作区必须能自举、读取与重建有效记录。create检查全部条件，open只接受文件中持久几何及允许的运行覆盖项。[持久格式](persistent-format.md)是具体排列、最小缓冲和恢复算法的权威。profile1固定要求FILE_BARRIER=3用于allocator提交；无达到该阶段的后端不能创建/打开持久模式，显式payload persist可以请求更强的受支持阶段。

## 7. 代表性算例

下表G=S=64KiB，均为公式结果，没有reserve/commit实测。32位仍先套用地址上限，不可使用后两列超出其上限的配置。

| 项目 | 16MiB | 1GiB | 4TiB |
| --- | --- | --- | --- |
| VOLATILE初始metadata | 64KiB | 64KiB | 64KiB |
| VOLATILE metadata上限 | 1MiB | 32MiB | 128GiB |
| ctx上限 | 16 | 1024 | 1024 |
| private span全局上限 | 4MiB | 256MiB | 1TiB |
| empty span全局上限 | 1MiB | 64MiB | 256GiB |
| 持久runtime R | 1MiB | 64MiB | 256GiB |
| WAL J | 256KiB | 8MiB | 64MiB |
| checkpoint每份K | 256KiB | 16MiB | 64MiB |
| 持久N | 1024 | 87040 | 1047552 |
| 持久固定保留F / B | 1966080 | 109182976 | 275079364608 |
| 持久payload候选C−F / B | 14811136 | 964558848 | 4122967146496 |

16MiB持久固定区为1.875MiB、payload候选14.125MiB；这是地址几何，不是1024个任意大对象保证，也不是全部工作区立即committed。16个ctx的64位记录成本为26624B（每ctx+heap1664B），另需pool页取整、索引、span及其他records。4TiB默认N约百万，不能宣称默认允许数十亿持久小对象；应用必须创建时增加CP/runtime预算并通过预估。

## 8. 维护与压力策略

不创建后台线程。普通申请失败前最多完成一次配置预算内的本线程drain/central回收轮，并重新搜索一次；不等待休眠owner。普通维护有对象数、页数、步骤数三维上限；span处理消耗步骤并按实际覆盖页数收费，OS子动作各消耗步骤，不够一个完整工作项则跳过并报告剩余提示。任何为0的维度禁止涉及该维度的工作，steps=0不执行维护；返回已做数量。RT扫描另由pool K约束。预算限制库发起的工作项数量，普通锁、OS调用或I/O的墙钟时间不受它保证。

线程flush只能处理调用者拥有的heap；arena maintain可在shared heap自身锁下维护该共享heap，并处理central/orphan和已退休资源，不可代替活owner访问私有freelist。shared、中央和退休工作共用本次三维预算，按并发协议分配有限切片；不足以覆盖各类完整工作时不承诺类别间无饥饿。RT维护使用pool固定K，不借用普通维护扩大一次扫描。持久checkpoint单独调用或WAL满时触发，复杂度随配置目录N/H变化，不放入声称有界RT的帧内操作。

普通维护在整个调用内共用同一份预算：检查一个class/owned-ring候选消耗step，接收一个remote/backlog节点消耗object，span迁移/退休及后端页操作才按实际覆盖页数消耗pages。仅drain对象不要求max_pages>0；但零页预算不能偷偷执行整span的物理回收。每次访问后保存heap类游标并轮转该类环头，小预算从上次位置继续，不能每次从class0开始；既有控制字段的同宽用途见S-01。中央和退休责任采用循环FIFO，预算不足/可重试失败移至队尾，单轮不反复选择同一失败记录。各维具体步骤及公平性前提见[争用与吞吐](contention-throughput.md)。

preflight与最终校验之间发生计划变化时最多内部重算一次，再次变化返回BUSY且尚未修改用户范围；额度耗尽走一次既定维护及一次重搜，不转成无限循环。公共选项仍为profile1的12项额度和6项策略，本轮没有新增在线自调限额、隐含后台工作或系统堆回退。

验证需要覆盖默认值重算、显式0、奇数CP总长度、溢出、两个并发quota tickets、替换双份峰值、增长额度互等、空span超额仍可free以及不同defaults_version的拒绝。具体证据由V-33和后续实现任务产出；本阶段只核对上述算术及合同。
