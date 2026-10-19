# 状态机、唯一完成责任与可重放反例

状态：设计候选，未实现、未执行模型检查或故障注入。本文件补充 [操作协议](operation-protocols.md)，把并发可见性、回退和清理责任写成转移规则。公开错误以 [API 契约](api-contracts.md) 和 [C ABI](c-abi-spec.md) 为准；持久事实以 [持久格式](persistent-format.md) 为准。

这里的状态名称是协议用语，不是一套新增 ABI 枚举。运行时仅使用 [S-01](metadata-layout.md) 已列明的 state、flags、effect、pin、链和计数字段；分析时的状态组合不物化成额外矩阵、全局原子、事务历史或区外分配。后续内部编码必须在这些字段内完成并校验合法组合；实际布局、原子宽度和后端行为仍须实施验收。

## 1. 分开描述状态，联合检查不变量

| 分析维度 | 权威记录与保护 | 可表达的事实 | 不能推导的事实 |
| --- | --- | --- | --- |
| 实例生命周期/准入 | ArenaHeader.health、生命周期保护；持久 control 的 admission/active | 可进入、关闭中、故障只读、入口已撤销 | 所有旧操作已完成、所有映射已解除 |
| 地址处置 | broker 中 ExtentDesc 的 owner_kind/substate/pending | 用户 owner、FREE、METADATA、独占 Pending、隔离 | RAM 已驻留、OS 仍保有同一 native reservation |
| 对象生命周期 | owner 的 slot/live；Large/Mapping/RT 状态 | 用户可用、已归还待接收、退休、已失效 | 后端费用已释放、descriptor 可立即重用 |
| 范围事务阶段 | RangeTxn.state/flags、active_next | 准备、独占、执行、补偿、发布、终态证据 | 提交持久化、其他线程可接管执行 |
| 后端效果 | TxnStep.effect 与 old/new BackendUnit | 无效果、已改变、已恢复、结果不明 | 整个 API 已成功、原内容仍完整 |
| 配额责任 | LimitBlock 与同步调用栈 ticket | reserved/used/已确认释放 | 该地址可分配、资源物理消失 |
| 持久事实 | commit 屏障、WAL/checkpoint、格式代次 | 未提交、确认提交、提交未知、恢复判定 | runtime 已完成发布、应用 payload 事务原子性 |

允许且必须正确处理的组合包括：逻辑 free + 仍有 backing + used 未减少；持久 commit 已确认 + runtime 尚待 apply；arena WRITE_FAULTED + 某个先前已确认事务仍需完成 apply；控制岛仍可访问 + 原 arena handle 已撤销。

禁止的组合包括：同一范围同时进入用户 FREE 与 Pending；尚未确认失去 backing 就返还其额度；control pin 非零而复用 descriptor；Unknown commit 被当成“未提交”撤销；RT Returned 已发布而释放者继续读 descriptor。

Pending/QUARANTINED 是 broker 的排他处置状态，不另造一份可与地址所有权竞争的账本。Pending 保留先前 owner、目标 generation 和后端责任；持久 R 内的 Pending/QUARANTINED 始终属于 METADATA 域。隔离只取消分配器对该范围的复用与用户访问授权，不承诺所有平台均能把其物理保护改为 no-access。

每条转移必须同时满足：地址半开区间不重叠；责任有唯一持有者；真实及未知后端资源均有记录和额度覆盖；旧身份未被复用；所有后续发布/失败报告所需空间已经存在。一个状态标签本身不是这些条件的证明。

## 2. Pending 的唯一完成者

### 2.1 状态与合法转移

首版同步 API 的调用栈是正在执行事务的唯一执行者。RangeTxn 及其 pin 保证资源可达，active list 便于受保护的观察；列表可见不构成帮助执行权限。无需向 RangeTxn 添加 executor 指针或线程 ID。

| 阶段 | 可进入条件与动作 | 唯一允许的后继 | 对其他调用可见的结果 |
| --- | --- | --- | --- |
| PREPARING | 输入/报告检查，取得完整记录、quota、输出空间；尚未取得目标 | ABORTED 或 PENDING | 目标仍可被竞争者改变，计划可能过期 |
| PENDING | broker 下重验身份/几何/信用，pin 目标和后端，挂 active 链，递增 begin revision | EXECUTING；无 OS 操作时 FINALIZING；已知无效果时 COMPENSATING | 范围不能再次分配；重叠修改 BUSY/COLLISION，分页 BUSY |
| EXECUTING | 同一执行者锁外进行计划中的 native 动作，逐步记录结果 | COMPENSATING 或 FINALIZING；持久路径可进入提交判定 | 仍为 Pending；已发生 OS 效果不等于 API 发布 |
| COMPENSATING | 尚未跨不可撤销提交边界，按依赖逆序补偿已作用步骤 | FINALIZING | 仍为 Pending，不能在每一步恢复后提前放入 FREE |
| FINALIZING | 不再发起新的业务 native 动作；已有记录足以描述全部片段；仍Pending/pin时在pool锁结算全部资源账 | COMMITTED、RESTORED 或 QUARANTINED | 资源已结算后，broker 内一次发布最终 owner/索引/身份，再撤 Pending 并推进 revision |
| COMMITTED | 请求语义已完整建立；必要持久提交已确认 | 归还事务临时记录 | 按 API 返回 OK；后端退休责任可能继续存在 |
| RESTORED | 原 owner、backing、保护及规定内容均已证明恢复，或完全无效果 | 归还事务临时记录 | 返回原错误；原合法对象/映射继续有效 |
| QUARANTINED | 至少一片不能满足请求或原合同；完整结果已发布 | 终态证据保留；后续独立清理可接管 | PARTIAL_FAILURE，隔离范围不可复用；不是仍执行中的 Pending |
| ABORTED | 未建立业务 Pending，归还本次预付资源 | 结束 | 用户范围无副作用；独立 metadata-growth 的合法结果可以保留 |

COMMITTED/RESTORED 是事务终态名称，不代替持久模式的 commit 事实。纯内存 map 的 COMMITTED 仅表示完成发布。对持久 Unknown commit，终态处置遵循第 5 节，不走普通“恢复原 owner”的分支。

事务进入 FINALIZING 前，所有可能失败、需要新配额或 native 调用的工作已经结束。保持 Pending 与 pin，在 metadata_pool 保护下把资源信用 reserved→used、归还多余信用并扣减已确认释放的旧资源，再进入 broker 检查自身仍是相关 extent 的 Pending 责任、target generation 未变、pin 仍有效；这些是既有独占责任的内部断言，不能变成已结算后再次尝试业务操作的入口。发布本身不得申请新元数据、执行 OS 或调用用户代码；否则会在无法保证回退的位置引入新的失败源。

完整结果发布、索引更新、有效身份变化与 Pending 撤销处于同一受保护的临界区。唯有 Pending 名额作为本次独有控制信用保留在栈 ticket.reserved，直到撤 Pending 后才退还。新资源的 used 必须在发布前建立，不能让另一线程看见新 owner 并退休它时从尚未建立的 used 扣账；“used+reserved 总数保守”不足以证明这个竞态安全。输出新 handle/指针、把 slab 供 heap 分配、交出退休清理责任均发生在资源结算之后。

### 2.2 停顿、取消与超时

- 执行者在 OS 内阻塞、被调度暂停或暂停于补偿步骤时，Pending 继续持有地址与资源。其他线程只可在不依赖该范围的操作上前进。
- watchdog、maintain、destroy、checkpoint 不得因时间已过而清 Pending、返回其额度、关闭其 native handle，或代替原执行者发布结果。一个步骤看起来“很久没变”不能证明 OS 不再使用其参数或资源。
- 首版不支持在分配器调用中异步终止线程、longjmp 越过栈、取消后丢弃正在运行调用，也不提供抢救该调用的原地接管 API。应用必须让调用完成后在安全点停线程。进程终止后的持久恢复是另一条离线恢复协议。
- 超时指标仅可作为上层诊断；现有 API 没有把一次内部等待超时转成“失败且无效果”的合同。销毁状态机的准入守卫不能越过仍活动的事务。公开 destroy/close 仍要求调用方先满足 quiescence，不把与正在执行调用并发销毁当成受支持的 BUSY 查询；应用已停止调用后，尚有 live 对象等前置资源时才按 API 返回 BUSY。
- 只有执行者已完成最终发布并交出退休/隔离责任之后，维护者才能通过对应控制锁领取后续清理。交接之前不能同时存在原执行者与“帮助者”。

因此，对不相交范围的并发可以继续；对同一待完成范围，不以破坏正确性的方式保证进展。普通操作没有 OS 墙钟上限。持续争用导致的计划重算另受 [操作协议](operation-protocols.md) 的一次内部重试规则约束。

### 2.3 control pin 的精确定义

| 阶段 | 必须满足的 pin 规则 |
| --- | --- |
| 取得 | 在能证明记录仍 live 且身份匹配的保护下增加 pin；检查 u32 上限，不回绕；新操作无法取得 pin 时在副作用前失败 |
| 锁外交接 | RangeTxn 持有 target、相关 extent、old/new backend 及必要共享父后端的责任；父 section 不能因为某个 view 已解除就过早关闭 |
| 读查询 | pin 只保证记录存储未回收，不保证字段不变；可变字段在其保护下复制成值后使用，不能带着 pin 无锁追踪变化中的链 |
| 修改 | Pending 负责排斥冲突修改，pin 负责生命周期，两者不能互相替代；调用方 quiescence 负责 payload 不被并发访问 |
| 结束 | 先完成发布、报告值构造和终态证据/退休责任移交，再放掉本次 pin；最后一个 pin 撤销后不得继续从旧记录读错误、统计或链指针 |
| 回收 | 从所有可达索引/链撤销，并确认 live/owner/pin/后端引用规则均满足后才能归还 pool；记录地址重用不能使旧指针重新有效 |

pin 不参与合法小对象 fast path 的全局引用更新，也不把普通 free 改造成需要新记录的操作。现有 live 协议保护合法 free；无对象所有权的查询必须遵守控制面 pin 规则。

首版每个 MappingDesc 同时只允许一个改变片段目录、保护、backing 或长度的业务事务，即使请求范围不相交。最终校验在 broker 下用既有 MappingDesc.state 取得变更权并 pin，后来的同 handle 变更返回 BUSY；释放变更权与最终目录发布一致。不同 MappingDesc 和不相交的新范围仍可并发。这样不会在预取完成后因另一事务改变 fragment_count/片段目录而使最终发布突然需要新记录。pin 本身不代替这份变更权。

map_persist 等锁外读取既有 backing 的控制调用，先在 broker 下确认无冲突 Pending/变更权并取得相关 pin；存在冲突则 BUSY。变更事务最终预检遇其他调用的冲突 mapping/backend pin 同样 BUSY，区别其自身事务 pin；不能在旧 flush 尚在使用地址时 replace/unmap该单元。broker 内完成值复制的普通查询不向调用方授予长期 pin。

## 3. 每个后端步骤的效果与补偿

### 3.1 四态 effect 和内部 flags

TxnStep.effect 保持已有四种语义：N=未执行/已证明无效果，A=已改变，R=已恢复，U=不确定。调用正在进行时也用 U 表示尚不能确定其最终效果；是否仍有执行者必须结合 RangeTxn 活动阶段判断，不能见 U 就接管。

既有 TxnStep.flags 的候选内部编码仅表达计划性质：REVERSIBLE（有经验证的完整逆操作）、CONTENT_LOSS（可能不可逆丢内容）、IDENTITY_SENSITIVE（重试要求证明 native 身份未变）、DURABLE_BOUNDARY（跨越持久提交判定）。REVERSIBLE 与 CONTENT_LOSS 不可同时宣称成立，除非计划已计费并保留足以恢复全部规定内容的独立 backing；此时 CONTENT_LOSS 描述的是原 native 子动作，REVERSIBLE 描述的是完整补偿计划。IDENTITY_SENSITIVE 可与任一项组合；DURABLE_BOUNDARY 不授予 commit 后补偿权限。具体 bit 编号在后续内部头文件中冻结，不新增字段或公开 ABI。

flags 在预检计划阶段由唯一执行者写入，Pending 发布后只读。effect/native_error/completed_count 在锁外只有唯一执行者读写；其他线程不得靠持 broker 锁就读取这些锁外可变字段。执行者完成后，在 broker 保护下发布终态值/报告；运行中诊断只读该锁保护的事务阶段、目标和 pin 信息。需要逐步诊断快照时只能由执行者在重新进入 broker 的安全点复制，不能偷读当前 OS 调用栈。

| 事件 | effect 变化 | 是否可直接对外发布 FREE/成功 |
| --- | --- | --- |
| 尚未调用 | N | 否，计划仍可失败 |
| 开始一个 native 动作 | N/A/R → U，保存调用前已知事实于既有 old/new unit 和栈内步骤位置 | 否，正在执行 |
| 返回且证明无效果 | U → 调用前确定状态，首次调用可回 N | 否，由整体协议决定 |
| 返回且证明目标动作已发生 | U → A | 否，仍有后续步骤及发布 |
| 返回但无法确定效果 | 保持 U | 否，进入保守补偿/隔离或持久故障判定 |
| 对 A 的完整逆操作确认成功 | U → R | 仍等全部相关步骤结束，不能把局部恢复当整体回滚 |
| 补偿调用已知无效果地失败 | U → A；原来就是未知则仍 U | 不能标 R，保留已改变事实 |
| 补偿调用结果不明 | 保持 U | 不能继续假定旧 backing/保护仍存在 |

`completed_count` 仅表示本轮已完成记录的进度，不是“前若干步一定已成功”的证明。错误可以发生在发生副作用之后；执行与补偿均逐步读取 effect。不能仅用 completed_count 决定回滚范围，也不能把一次错误返回自动改写为 N。

### 3.2 依赖、不可逆动作和回滚再次失败

1. OS 前把依赖安排成确定的执行顺序，逆序即满足补偿依赖。需要复杂资源关系时使用已计费 old/new BackendUnit、父引用和 TxnStep 次序，不临时建立外部 DAG。拆成更多独立 native 子动作时，先增大 k 并重新核对 E/B/J、backup、quota、报告上界；不得假定原 `3k+2` 自动覆盖任意新片段。
2. 每个可能失败的前缀必须已有可表达的收口：完整逆操作，或者已知最终片段与隔离责任。preflight 只证明“有补偿方法和资源”，不承诺补偿 OS 一定成功。
3. replace、discard、匿名 unmap 可能丢失原内容；old protection、file offset 或 descriptor 不是内容备份。只有独立保留的原 backing、明确平台内容保留能力或已计费完整副本足以支持 ROLLED_BACK。对 PRIVATE/COW，不得从重新 map 同一文件推导私有修改仍在。
4. 补偿每个子动作至多执行一次本轮尝试，按逆依赖推进。补偿失败不递归发起“回滚这次回滚”，不临时申请更多记录或无限重复同一 native 调用。可继续处理与失败资源独立、且前提仍可证明的其他责任；依赖已未知状态的动作跳过并保留责任。
5. 一旦已知完整请求成功且完成对外发布，不把后续清理失败变成撤销用户对象；保留退休责任。持久 commit 已知成功则先完成 apply，禁止反向撤销。commit 未知也禁止按未提交补偿其 durable ownership。
6. 最终分类来自实际事实：请求效果已建立为 APPLIED；原状未动为 UNCHANGED；原合同完整恢复为 ROLLED_BACK；既不能授权请求状态又不能证明原合同的片段为 QUARANTINED。最后一类包括“已知内容损失、只有安全 reservation”以及 native 结果未知，不必虚构两者都不可观察。

已确认恢复的片段、已应用片段与隔离片段在同一次最终发布中给出不重叠的完整报告。任何未知资源继续占额度；后续独立清理成功可以收回空间，但不能追溯把上次 PARTIAL_FAILURE 改称无效果。

## 4. 逻辑释放、退休与物理清理

| 转移 | 唯一责任与观察规则 | 可否因清理失败撤回 |
| --- | --- | --- |
| 小对象 LIVE → 本地可用 | owner 更新本地链/live；这是对象释放点 | 不可 |
| 小对象 LIVE → REMOTE_PUBLISHED | 合法释放者 release CAS；owner 尚未消费前仍贡献 live | 不可；发布后释放者不再访问 slot/descriptor |
| REMOTE_PUBLISHED → owner backlog → 本地可用 | 单 owner acquire exchange，按预算消费；处理时才减 live | 不可；未消费链由owner持有，exchange至backlog落位间可暂存执行中调用栈，lease阻止接管 |
| 空 slab/large/完整 page lease → RETIRED | 撤销用户/分配索引，既有 descriptor.retire_next 移交责任 | 不可；不足 RangeTxn 只延后清理 |
| RT LIVE pool → RETIRED | quiescent 且无 Allocated/Returned，撤 binding/索引，原 RP/RC 接管 | 不可；destroy OK 后旧 pool 禁止再调用 |
| RETIRED → CLEANING | 维护者在正确控制锁下独占领取，取得已有记录/预算后建立清理 Pending | 无资源则保持 RETIRED；不得复活对象 |
| CLEANING → FREE | 后端确认兼容 reservation/借用 backing 满足该域复用条件，再由 broker 发布 | 此时才允许新 owner 取得范围 |
| CLEANING → RETIRED/QUARANTINED | 已知仍原样可重试则退休；结果不明/不可授权则隔离 | 用户对象仍已释放 |

REMOTE_PUBLISHED 后，sanitizer 标注、对象来源统计和错误字段采集若需 slot/descriptor，必须已经完成。发布后仅允许访问释放线程自身有效的 context/栈和预先复制的值；把 bookkeeping 放在“不是算法主体”的函数里不改变该限制。

orphan 移交要求旧 owner 最后一次 owner-only 访问先于 central 释放 ownership lease，新 owner 获取同一控制保护并独占claim后才取得转移责任。claim期间仍为ORPHAN；释放central后，在目标heap有效lease/锁下安装owned环和完整状态，最后release发布新owner token，不能先公开shared token再安装状态。发布新 token 不是允许活跃旧 owner继续访问的许可证；睡眠 owner 不等于 orphan。控制 pin 可以阻止退休，不能赋予第二个 consumer 权限。

### 4.1 终态隔离证据如何被维护找到

终态 QUARANTINED 从 active Pending 链退出、pending_count 减少，随后归还原 Pending 名额；它不再是仍运行的业务调用。X/J、未决 BackendUnit 及相关 pin/额度仍保留，不能一起归还。

- 每个相关 Q ExtentDesc.pending 指向终态 X；Q 状态下 free_prev/free_next 复用为 X.cleanup_list 的双链。该 extent 不在 FREE 索引，原 owner_prev/owner_next 仍可服务 mapping 片段链，两类链不混用。
- X 退出 active 链后，active_next 清空并复用为 retire_tail 循环 FIFO 中的 next。退休队列根据 prefix.kind 读取 X.active_next 或其他记录的 retire_next；同一字段绝不同时属于两个队列。X 的孤立 backend 责任继续由 steps.old_unit/new_unit/父引用保留，cleanup_list 只包含 Q extent。
- 原执行者先复制报告、错误和最后需要的值，再交出 X；一旦其进入可被维护者领取的退休队列，原执行者不再访问 X/J/关联对象，只能收尾自身栈 ticket 等独有状态。维护者领取期间 X 不在队列，其他线程不能重复取得这份责任。
- 每次 arena maintain 对 central orphan 与 retire 各最多领取 8 项，一类只开一个 batch；在 central 下独占摘取到固定 16 指针栈工作区，解锁后处理。受 steps 预算限制可少取，两类队列及 shared 冷类访问的服务次序、预算和条件性进展统一遵循 [并发协议](concurrency.md)。未完成项批末入尾，本次调用不再领取；不持有未经 pin 的“初始 tail”当一轮结束哨兵。
- 对终态 X 进入后续清理前，在 broker 下独占其责任并按需建立新的清理 Pending；X.active_next 此时才可转作 active 链。前一业务执行者已经退出，不是帮助仍运行中的事务。没有记录/信用或工作预算则保留终态责任并在批末重排队尾。
- cleanup_list 中确认恢复复用条件的 extent 在 broker 下摘链、清 pending 引用，再按真实最终 owner 发布；仍不确定的留在链中。只有链空且所有步骤/后端责任已解除或明确移交给其他既有 owner，才可回收 X/J。不能只因第一个 extent 已成功就回收共享事务。
- completed_count 在进入终态清理扫描阶段时明确重置，之后只作为步骤扫描游标；步骤扫描与 extent 链处理以既有 RangeTxn.state 区分阶段，逐项计入 max_steps。extent 从 cleanup_list 头部处理，未能推进者轮转，不能在预算外遍历整条链计算长度。空链判断为常数时间，后端是否收口依逐步 effect/责任检查，不能把游标当成功数。

步骤扫描、native 动作、extent 发布和队列重排都计入维护预算；预算不足的领取项仍需用预留的固定 bookkeeping 步骤归队，不能泄漏在失效栈中。一次 native 动作按其整个不可拆单元计 pages/steps，预算不足就跳过。终态 X/J 的容量来自原事务信用，本方案不增加 refcount、队列节点、背景线程或无界地址树扫描。

## 5. 持久提交与 runtime 发布

| 持久事务阶段 | 允许回退的内容 | 必須完成的责任 |
| --- | --- | --- |
| 尚无 WAL/commit 效果 | 未公开新 backend 可按普通逆序补偿 | 原持久对象与根不变；退还已确认未占用资源 |
| redo 已写但 commit 已证明未发生 | 可清理本次临时 runtime 资源；旧 payload 不得曾被 free 改写 | 后续格式只承认有有效 commit 的日志；不把“可能写入”当已证明未提交 |
| commit 屏障已确认，尚未 apply | 不再反向撤销分配、free 或 root | 用预付记录完成 runtime apply/publish，再结束 admission；不能临时 metadata-growth |
| commit 结果未知 | 不得按未提交回滚，也不得把目标空间交给其他请求 | latch WRITE_FAULTED，保留恢复证据与未决责任；返回 COMMIT_UNKNOWN，重开判定 |
| runtime apply/publish 已完成 | 不因后续 decommit/close 清理失败撤销持久事实 | 回收转独立责任，结果及健康状态分别报告 |

WRITE_FAULTED 是实例故障与新写入准入状态，不能覆盖事务此前已确定的阶段。后端准备、redo、commit、主文件 persist 各有限阶段由既有 admission/control 短锁签发许可；fault latch 后不发新许可，之前签出的一个有限阶段可以实际执行并分类，不能声称 latch 瞬间停止了所有 OS。commit 许可覆盖记录写入及全部规定屏障，不在写过 commit 后因下一屏障未获许可而假称未提交。

另一个 data persist 失败与某事务 commit/apply 并发时：从未取得 commit 许可且能证明未提交者保留旧 durable ownership，按 WRITE_FAULTED/已知 IO 错误退出；commit 已确认者仍保持 Pending/pin 先结算资源、再 runtime apply/publish，返回本次 OK 且健康状态可能为 WRITE_FAULTED；已发 commit 但事实未知者返回 COMMIT_UNKNOWN，禁止按未提交反向补偿。具体状态码及健康交错处理由 [持久设计](persistence.md) 定义。

主文件 persist 与持久修改纳入同一 admission 排空范围并 pin 目标，close/checkpoint 不能只等拿 commit gate 的事务；pfree 最终校验发现该 persist pin 时返回 BUSY。WAL 满请求先退出自身 admission 再请求 checkpoint，避免等待自己。checkpoint 使用稳定 runtime 截点，必须等已确认 commit 的 apply 完成，也必须在故障状态下停止覆盖恢复证据。

第一份新super确认后，若第二份镜像已证明无效果地失败，进入内部CHECKPOINT_HELD：health仍READY，但admission关闭，新持久修改BUSY；既有recovery_phase/admission_open/manifest字段保留责任。显式checkpoint在关闭的admission及已排空的在途集合下唯一claim；正常close另须调用方quiescence，不能用内部admission代替业务停用。两者继续未完成的第二镜像，不重做第一半、不另起新checkpoint、不无限内联重试；abort-close可以退出。第二镜像确认前不复用旧WAL/CP，重试结果未知转WRITE_FAULTED。这个中间态不新增health值或布局字段。

CLEAN_REOPEN 的每次业务修改只有 runtime 发布点，没有逐次 durable commit 保证；正常 close 的 checkpoint 与双 super CLEAN 发布是独立持久边界。其 runtime 状态机不能冒用 RECOVERABLE_METADATA 的异常退出承诺。

## 6. 清理 token 与身份安全

### 6.1 销毁入口与最后控制岛

| 阶段 | arena handle | cleanup token | 可做的事 |
| --- | --- | --- | --- |
| 前置检查不满足 | 仍有效 | EMPTY | 返回 BUSY，调用方先停止访问和完成资源生命周期 |
| 持久 close 的 checkpoint/clean 阶段失败，尚未拆 runtime | 仍是诊断/abort-close 入口，未知时 WRITE_FAULTED | EMPTY | 按持久故障合同处理，不能假称已清理 |
| 已开始不可逆 runtime/native 拆卸 | 不再接受普通调用 | 内部固定清理状态；未收口则输出非空 token | 仅推进已拥有资源的清理，不回到 READY |
| 仍有 cleanup-only 控制岛 | 失效 | CONTROL_ISLAND | 唯一 token 持有者调用 retry；不可复制并发调用 |
| 最后控制岛待释放/结果不明 | 失效 | 最后所需固定信息已转入 token | 不再解引用可能已解除的控制内存，包括错误和统计输出 |
| 全部责任已确认结束 | 失效 | EMPTY | 成功结束；旧 token 的副本不能再次清理 |

每个成功关闭的 handle/view 或释放的单元立即从可执行责任中撤销，再进行下一步。先撤 parent section 或控制岛、后读其子责任是非法次序。未知诊断可以阻止 token 清空，但不能使已完成步骤再次执行。

### 6.2 幂等性的边界

`cleanup_retry` 的幂等来自保存“剩余且身份仍可证明”的责任，不来自 native 操作名称。只有后端证明重复执行不会作用于另一个新 owner，才允许重试。原生句柄值、相同 base/length、一次地址查询或 token nonce 均不是所有权证明。

典型反例是 close(EINTR) 后同一 FD 已被别的线程复用；另一个反例是 reservation 恢复失败形成洞，外部映射进入同一地址。两者都不能通过再次 close/unmap 同值来“确保清理”。broker 中隔离该 arena offset 不代表 native 地址仍属于本库；只能保留 UNKNOWN_NATIVE_ID 等既有诊断，继续处理独立的已知责任。禁止用整段 unmap(C) 覆盖包含身份不明洞的范围，也不能用 fixed replacement 抢回可能属于第三方的映射。

token 最后的固定责任无法证明已结束时，不伪造成功或清空 token。应用可停止进程或作离线排障，但当前实例不得重新开放分配。这个边界不新增 token 字段，也不提供绕过身份验证的强制清理 API。

一般业务调用也不具有“看到错误就原参数重试”的幂等性：普通 free/完整 page_free/RT destroy 成功后旧身份已失效；PARTIAL_FAILURE 先读完整报告再对仍拥有片段发新请求；COMMIT_UNKNOWN 先重开查询 allocation identity/root；capacity 预估重试重新验证计划。operation_id 仅诊断，不是首版去重键或历史结果查找键。

## 7. 可重放反例与后续验收

下面是未来可控 VM 模型/调度器的测试剧本，当前只规定输入、暂停点、故障及应拒绝的执行轨迹，未运行。每次重放记录 seed、线程、operation ID、RangeTxn/target generation、范围、原生动作序号、注入结果、effect 与最终 owner；日志由测试环境承载，不给生产 fast path 增加日志。

### V-44：状态积与双重失败

1. T1 替换由两个 view 构成的映射，完整预取 k、旧 backing、报告和信用。暂停于第一个子动作已改变、第二个子动作即将调用。
2. 第二步注入“返回失败但 native 效果未知”。T1 进入逆序补偿；再向第一步的逆操作注入“已发生部分效果但返回失败”。
3. 调度 T2 对目标申请 fixed map、执行 query；T1 未收口前目标不可复用，分页 BUSY。模型另检查销毁准入守卫拒绝转移，不通过并发调用公开 destroy 来探测活动事务。
4. T1 发布 APPLIED/ROLLED_BACK/QUARANTINED 的实际片段；未知项不返额度。再次维护只能依据现存责任推进，不能执行一次无记录的“回滚回滚”。

判据：每个 OS 边界及每种故障分类可重放；错误码不决定效果；任何时点不双重所有；全量报告不缺片；old descriptor/protection 不能被当作 payload 备份。增加 PRIVATE/COW 原内容不可恢复和已知内容丢失的变体。

### V-45：唯一完成者、暂停与查询 pin

1. T1 建立 Pending，pin 目标/old/new backend，在 native 调用入口暂停。
2. T2 maintain、T3 查询依次运行，并模拟 watchdog 到期及模型内销毁准入守卫。它们不得取得 T1 的执行权、释放记录/信用或假造完成；不相交范围仍可由其他调用使用。公开 destroy/close 不与 T1 并发调用。
3. 恢复 T1，令 native 成功，再暂停于 publish 前。重复步骤 2，确认 OS 成功没有提前解锁范围。
4. T1 发布后查询持 control pin；调度最后一次退休并尝试回收/reuse 同一 descriptor。pin 释放前记录不得复用；pin 后查询不得再读它。
5. 以两个事务交错 begin/end 和 revision 近耗尽替代单事务，验证 begin/end 信用、饱和后合法清理及分页 STALE_CURSOR。
6. T1 完成后让全部线程停止调用，在仍有 live 对象/ctx 等前置资源的情况下顺序执行公开 destroy，按 API 验证 BUSY；与活动调用并发销毁仍是调用方违约，不列作支持承诺。

判据：一个运行中的 Pending 恰有一个完成者；查询看到受保护的值副本；无强抢、无 use-after-free、无 revision 倒退；停顿不被错误包装成无效果超时。

### V-46：清理的身份与单向移交

1. 令 create 的 bootstrap 失败，reservation 清理也失败，检查固定 PRE_CONTROL token 能覆盖全部已取得责任；没有提前取得任意文件/views。
2. 令已运行实例拆卸到最后控制岛时解除结果不明；在模型中把控制内存设为不可访问。错误输出/retry 不得读取它，旧 arena 入口不得复活。
3. 注入 close 后 FD 已关闭并被其他线程复用；另行注入 reservation 洞被外部 mapping 占用。retry 不得触碰两个新 owner；其余明确自有资源仍可清理。
4. 重复合法的单一 token retry，成功资源不重复执行；复制 token 或旧 handle 调用属于接口违约，测试调试诊断但不宣称可容错。

判据：可执行清理责任单向减少，未知身份不变成重试地址，parent/control island 最后释放，未知责任未收口不宣称 EMPTY/成功。

### V-47：成功点之后的不可撤销责任

1. remote free 在 release CAS 后立即暂停，owner 消费最后节点、撤索引并回收 descriptor；恢复生产者，只许访问自身有效 context/栈，sanitizer/error/stats 不能访问旧 slot/descriptor。
2. 完整 page_free 或 RT destroy 已逻辑 OK，但无 RangeTxn 或 OS 清理失败；保持 retired/quarantine 责任且旧身份失效，不能以清理错误撤销成功。
3. 持久事务 commit 屏障确认后暂停；另一主文件 persist 注入未知错误使 health=WRITE_FAULTED。先前事务仍完成资源结算与 apply；checkpoint 不越过尚未完成 apply，也不覆盖故障证据。追加 redo 已发但 commit 未获许可、commit 整阶段已许可未返回的交错。
4. 把第 3 步改为 commit 结果未知。不得恢复旧 FREE 或据此再次分配；重开按有效日志判定，再通过 allocation identity/root 查询决定结果。
5. 在资源最终结算前暂停 map/slab refill；竞争线程不得先取得新 owner 并扣其 used。结算后发布，再立即逻辑释放/维护，核对每维额度不重复转账，Pending 控制信用只在撤 Pending 后归还。
6. checkpoint 首镜像确认、第二镜像已知无效果失败后，模型检查 CHECKPOINT_HELD 的 admission 关闭；显式重试只续第二镜像，未知结果停止写入，旧 WAL/CP 未提前复用。

判据：逻辑释放、broker publication、durable commit 和 native cleanup 四种成功点不混用；后续故障不能造成双重释放、已提交事务撤销或 stale handle 复活。线程 owner 移交另以 central 临界区顺序与 token 重用压力复核。持久许可、CHECKPOINT_HELD和恢复证据的跨层故障剧本另见 [故障恢复](failure-recovery.md)。

上述 V 编号及实施依赖由统一 [验证矩阵](validation.md) 追踪，不能因为本文件已有剧本就标记通过。Pending 唯一完成者、禁止强抢、效果四态、不可逆边界与单向清理责任的决策见 [ADR-035/036](decisions.md)。
