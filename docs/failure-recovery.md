# 故障分类、在途操作与恢复责任

状态：设计合同，尚无实现或故障注入结果。本文件统一 [操作协议](operation-protocols.md)、[平台后端](platforms.md) 与 [持久化](persistence.md) 的失败语义；公开状态码、health 和报告布局仍以 [C ABI](c-abi-spec.md) 为准，不新增 ABI 或磁盘格式字段。

## 1. 先确定事实，再选择状态码

一次失败必须分别回答四个问题：用户对象/映射的逻辑操作是否生效；原生地址和资源身份是否仍归本实例；内容和保护是否满足原合同；持久提交或数据屏障是否已确认。一个 OS 错误码不能代替全部答案。

| 可证明的结果 | 用户可见结果 | 范围和资源责任 |
| --- | --- | --- |
| 用户范围尚未修改，或后端证明调用完全无效果 | 对应参数、配额、OS 或 I/O 错误 | 原 owner 不变；只归还确实未消费或已释放的资源 |
| 用户范围曾变化，但地址、backing、保护和应保留内容均已恢复 | 对应失败，原合同继续成立；片段可报告 ROLLED_BACK | 回滚资源确认释放后退账，不能仅以“重新映射成功”认定恢复 |
| 部分范围确定生效，其余确定未变 | PARTIAL_FAILURE 与完整逐片报告 | 确定部分按实际 owner 发布；不能向调用方隐瞒已生效修改 |
| 某些范围、内容或原生身份无法证明 | PARTIAL_FAILURE，相关范围 QUARANTINED | 不参与复用；保留既有记录与保守 charge，只操作仍能证明归属的原生单元 |
| 普通 VOLATILE free 已逻辑完成，物理清理失败 | free 的成功不撤销；维护/诊断报告后端失败 | 对象已失效；已有 retire/隔离记录继续负责 backing，不允许再次 free |
| 本次持久事务确认未提交，但另一调用已将主文件置为 WRITE_FAULTED | WRITE_FAULTED；本调用自身已知 I/O 失败也可返回 IO_ERROR，并报告 health | 不输出新对象、不把未知事务当已撤销；既有 live 对象身份保持原事实 |
| 本次 RECOVERABLE_METADATA commit 已确认，runtime publish 完成 | OK；即使并发错误使 health 已是 WRITE_FAULTED | 成功事实保留；故障后的复用和新破坏性回收仍被禁止 |
| commit 已可能生效，但无法确认结果；或主文件 data persist 的所需持久阶段未知 | COMMIT_UNKNOWN，health=WRITE_FAULTED | 保留恢复证据，停止新的写入与复用；由 abort-close/open 和身份查询判定 |

失败前独立完成的 metadata-growth 可以留下可用内部库存；“业务操作无效果”不保证 OS 资源计数与调用前逐字相同。反之，rollback 不能用资源计数恢复来冒充对象内容已恢复。

## 2. VM 回滚的证据

后端为每个 TxnStep 记录未执行、确定无效果、已改变、已完整恢复或未知，并记录实际仍持有的原生资源。流程只允许一个 executor 推进 Pending；首版同步调用栈持有执行权，原有 RangeTxn、pin 和步骤记录保留责任。诊断线程不能因超时撤销 Pending、夺取 executor 或重发同一步 OS 调用。

完整回滚至少需要同时证明：

1. 目标区间仍被本实例独占，且没有释放后重获地址的窗口。
2. 原 backing 的身份、offset、共享/COW 属性与边界恢复。
3. 原保护和未授权改变的访问属性恢复。
4. 操作承诺保留的字节仍然存在；匿名页和 PRIVATE/COW 修改不能由“重映射同一文件”重新生成。
5. 相邻未涉及的 live 范围始终保持有效，没有为了修复中间洞而暂时解除整个大 view。

内容备份只有在操作合同明确需要、预检已预付峰值地址/commit/记录预算且能恢复原语义时才可使用；不能在 OS 已修改一半后临时借系统 heap 补做备份。不能证明完整回滚时使用逐片结果。显式 discard 所授权丢弃的内容不属于必须恢复的内容，但失败仍须说明究竟哪些页已丢弃。

POSIX 恢复保留态必须使用已验证、保持自身范围所有权的固定替换原语，不能采用 munmap 后重新抢占地址。Windows 使用满足精确 placeholder/view 边界的 preserve/replace 操作。平台 spike 必须确认真实失败语义；若合法失败可能产生失去 reservation 的窗口，不能把该原语启用为满足完整连续保留合同的能力。

## 3. QUARANTINED 不证明仍拥有原生地址

broker 隔离表示“本分配器不会把此范围再次交给用户”，不表示内核仍把每个地址授予本实例。若某次原生解除/替换的结果未知，后续同一 base/length 上的映射可能已经属于外部线程。这与被 OS 重用的 FD 数字具有相同的身份问题。

- 只凭地址相同、VirtualQuery/映射查询显示有页、FD 数字仍有效，都不足以重新证明原资源身份。
- 未知范围禁止盲目 MAP_FIXED、unmap、protect 或清理重试；销毁不能整段释放 capacity 来覆盖可能被外部占据的洞。
- 若能界定失证范围，控制结构和其余范围所有权仍可信，只隔离该范围。其他有效范围继续遵守原合同；VM 故障不借用 WRITE_FAULTED，也不新增 arena health。
- 若受影响范围无法界定，或控制结构完整性本身已失信，这是后端核心不变量破坏。停止使用该实例并尽可能保留诊断证据，不能宣称任意合法 free 仍可安全执行；它不属于已验证后端合法失败应提供的局部恢复保证。

cleanup 只处理后端合同证明仍由库持有的单元。可重试责任转交已有 cleanup token/control island；无法安全重试的身份只保留诊断，不能把猜测的资源写进执行列表。最后控制岛仍有未决责任时不得提前解除。borrowed buffer 始终由调用方管理原生映射，库不通过 cleanup 取得解除权限。

## 4. 逻辑释放与物理退账

逻辑 free、地址回到可分配集合、view 消失、commit charge 消失是不同事件。只有对应后端资源的释放得到确认，才能减少其 used；返回错误、改成 NOACCESS、解除一个 section view 或删除一个逻辑对象都不自动退还 commitment。

Windows pagefile section 的 retained commitment 由 section 生命周期和剩余 views 决定，不能按本次逻辑释放页数直接扣减。POSIX discard 的效果也不能冒充 Windows commit charge；统计只能报告该平台实际可证明的口径。

事务预付按峰值占用，已用和未知部分转为 used，多余且确定未消费部分归还。隔离记录不占有即将离开的调用栈 ticket。普通 VOLATILE free 使用分配时已备好的退休责任，不能因为后续 RangeTxn、quota 或报告空间不足而使已合法释放的对象复活。

## 5. 主文件 health 与阶段许可

持久化 admission、health 更新与阶段许可由实例现有持久 control 的 admission LockSlot 串行化；生命周期控制协调实例关闭，不新增锁槽，也不增加普通 volatile/RT 对象路径的共享原子。一次调用从进入 admission 到结果分类、runtime publish 和资源责任收口都计为在途，包括等待 commit gate 的调用。control 的 health 镜像与 ArenaHeader.health 在该锁下按同一事实更新，不能各自批准写入。

持久修改在 gate 外准备资源，取得 commit gate 后短持 broker 做最终身份/root/pin 校验与 Pending，释放 broker，再取得本次后端阶段许可。gate 持有者可以短取控制锁；控制锁内不得等待 gate、broker、OS 或任何在途调用。checkpoint/close 关闭 admission 后释放控制锁再等待，不能形成反向等待。

许可不是整次调用无限期写入的通行证。需要分别重查 health 的有限阶段为：新增 backing 的单次后端准备、redo 写入及其同步、一个事务的 commit 写入及所需完整屏障、一个 data persist 的所选屏障协议，以及 checkpoint 的受控发布阶段。阶段工作量在预检时已界定，许可不能用于下一事务或未计划的批次。

commit 阶段的一次许可覆盖本事务 commit 记录写入和完成判定所需的完整屏障；故障后不能因下一次 flush 没再获许可，就把已经写出的 commit 宣称未提交。后端可以把记录写入和屏障分为多次 OS 调用，但它们属于这一有限已发阶段。

CLEAN_REOPEN 没有逐次 WAL，其 runtime 变更同样须取得本次有限发布许可；故障前已许可并完成的调用按该模式的 runtime 成功合同返回。它不能借“已发布”获得 RECOVERABLE_METADATA 的崩溃提交保证。

WRITE_FAULTED 的锁存与许可签发在同一短锁下排序：

- 故障先锁存：不签发新的写入、复用或破坏性回收阶段；只做结果分类、已确认提交的 runtime apply，以及经证明安全的资源收口。
- 许可先签发：这一阶段已属在途，可能在故障锁存后才物理进入 OS；允许它完成并报告真实结果。不能承诺错误被观察的瞬间取消另一线程已获许可的系统调用。
- 一个阶段完成后，下一阶段重新检查。仅 redo 已发而 commit 从未获许可、可证明没有有效 commit 时，本事务确定未提交；其结果不能因另一调用失败而被改成 COMMIT_UNKNOWN。
- commit 确认后必须完成预留好的 runtime apply，不能因为 health 改变而回滚或遗漏该事实。apply 不进行新分配、不进入 OS；其自身不应因元数据不足失败。控制结构失信造成无法 apply 属于核心不变量破坏，不能重新解释为正常可恢复的已知失败。

已确认 pfree 可以在故障后完成逻辑释放并返回 OK，但其空间不得用于新分配，也不再启动 discard、清零、覆盖用户 payload 或新的破坏性物理回收。尚未消费的 ticket 可归还；已准备或未确认消失的 backing 保留既有记录与实际额度，待受控关闭处理。

资源结算先于最终 runtime 发布，不另增结算状态或 pin 字段。OS/补偿/提交结果确定后，保持原有 Pending 和全部 pin、commit gate/admission；在不持 broker 时短取 metadata_pool，将实际/未知资源从 reserved 转为 used，并仅对已确认释放的资源扣账。解 pool 后回 broker，以预留记录无失败地 apply/publish 并撤 Pending，最后释放或移交 pin、交付结果、退出 gate/admission。Pending 名额自身的 ticket.reserved 保守留到撤 Pending 后再退；其他资源必须先结算，避免新 owner 已可见但 used 尚未到账。gate→metadata_pool 只允许不申请新资源的有限结算，pool 不得反向等待 gate，pool 与 broker 始终不嵌套。

这里“禁止复用”约束整个故障 durable arena 的修改入口和维护路径；不能从 session map、普通对象 API 或后台清理绕过。其他独立 arena 的 admission、gate、预算和正常 free 不受影响。

## 6. Data persist 的 admission 与目标 pin

主文件 sa_persist 也进入 durable admission，在 broker 下复核 allocation identity、generation、范围和存活状态，取得已有 descriptor 的控制 pin，释放 broker 后申请 persist 阶段许可并执行 I/O。结果分类及必要 health 锁存先于退 pin 和退出 admission。

这个 pin 只保护本次同步调用所使用的映射和身份。pfree、backing 替换或退休的最终验证遇到冲突 pin 时返回 BUSY，不等待一个可能依赖自己的 I/O 调用。pin 不替应用保护任意业务读者，不允许调用方一边改写 payload 一边假定 persist 给出了业务快照。普通 ref_resolve/query 的返回也不隐含永久 pin。

主文件 persist 不必持有 metadata commit gate，因而允许与不冲突对象的元数据提交并行；它的错误使用第 5 节的共同 health/许可顺序约束并发事务。外部 session mapping 的 persist 固定该 mapping 的现存 backend 单元并受实例生命周期保护，但其数据失败不自动将无关主文件变成 WRITE_FAULTED；其结果也不提交 allocator root。

## 7. Checkpoint、close 与排空

checkpoint 先关闭 durable admission，再排空所有已进入调用：gate 等待者、data persist、后端准备中调用、commit 已确认而 runtime 未发布的调用都包含在内。等待期间不持 broker、控制锁或 commit gate。已有请求必须完成既定有限阶段并退出，不能因 admission 已关闭而永远等下一次进入资格。

排空后重新检查 health，健康才取得 commit gate 并生成一致快照。WRITE_FAULTED 禁止 checkpoint 覆盖、日志回收或“修复性 flush”；不能用当前 runtime 未决状态猜测磁盘事实。

checkpoint 发布许可覆盖一个有限的双 super 发布协议。若第一槽已确认发布新 manifest，而第二槽的后端失败能证明该次发布完全无效果，调用返回 IO_ERROR，health 保持 READY，但进入内部 CHECKPOINT_HELD：admission 保持关闭，新修改返回 BUSY。现有 control 的 recovery_phase、admission_open、dirty_state 和 manifest 计数保留完成阶段，不新增字段或公共 health 值。旧 WAL/CP 不得回收，不能退回旧 epoch 继续写，也不能宣称 checkpoint 已成功。

显式 sa_checkpoint 或满足 quiescence 条件的正常 close 可以在短控制锁下唯一 claim 此责任，按保存的 manifest 只完成尚缺的镜像；已有 executor 时返回 BUSY。不得重新写已确认的第一槽、另起一轮 checkpoint 或无限内联重试。镜像确认后才释放旧资源并开放 admission，close 则继续其原定关闭流程；若持有的是先前 DIRTY checkpoint 的半成品，先完成它，再执行 close 所需的 CLEAN 发布。任何重试产生未知持久结果都转入 WRITE_FAULTED。abort-close 可以直接退出当前会话，仍须保留已有磁盘事实。

该分支要求后端证明第二槽完全无效果，常规写入/flush 错误不能靠缓存中的 read-back 认定磁盘未改变。无法证明时一律使用未知结果合同。创建/打开尚未发布 arena handle 时没有供用户继续 checkpoint 的实例，半成品由初始化失败及 cleanup 合同处理，不输出 CHECKPOINT_HELD 的可分配 handle。

WAL 满的调用须在业务 Pending 和 WAL 写入前，先退出 gate、admission 并归还自身 ticket，再请求独立 checkpoint。checkpoint 协调者不能等待包含自身的在途计数归零。checkpoint 成功后原请求从头预检，不沿用旧身份、范围或额度报价。

正常 close 的调用方先停止新使用、排空业务访问并满足 ctx/session/临时资源的显式前置条件；库的关闭屏障再次验证没有遗漏在途责任。持久 live allocations 不阻止关闭。最终持久发布得到确认后，才能进入 runtime/native 拆卸。若尚未开始拆卸而持久阶段失败，原 arena 仍是诊断与 abort-close 的入口。

abort-close 不取消已经运行的 executor。调用方必须先停止使用并等相关操作完成结果分类；库排空 admission、pin 和 Pending 的执行责任后，仅清理会话 native/runtime 资源，不写新 checkpoint，不覆盖 WAL/super，不试图撤销已提交事务。文件 lease 与身份登记在全部可能写主文件的调用退出后才解除，控制岛最后处理。开始不可逆拆卸后，未清完责任由 cleanup token 接管，旧 arena handle 不再作为入口。

## 8. 未知 free 与恢复后的身份

COMMIT_UNKNOWN 不意味着“失败无效果”。同一个 pfree 可能已经落盘；如果立即重试，第二次可能释放错误代次、掩盖第一次结果，或与恢复所需内容发生冲突。当前会话禁止此类盲重试，也不能把旧指针重新当 live 对象写入。

应用保存已知 ref，停止该实例写入，abort-close 后重新 open，再按 arena UUID、allocation ID 和 generation 查询恢复账本：仍为该 live identity 时才由应用决定下一步；已 free/retired 或已有新 generation 时，旧 ref 不再拥有释放权限。无响应的 palloc 可能产生应用未记录的已提交对象，由 allocation 枚举发现；没有直接 root 仍不证明对象可自动回收。

## 9. 后续故障注入验收

以下是待实施实验，不是已通过结果，纳入 [验证矩阵](validation.md)：

- V-52：逐个 native 步骤注入无效果、部分成功、内容丢失、rollback 失败和原生所有权失证；证明报告完整、不会盲固定覆盖、不会整段 teardown 跨越陌生映射，只有确认的资源释放才退账。
- V-53：在主文件 persist 锁存故障前后交错 redo、commit 许可、commit 屏障确认和 runtime publish；分别得到确定未提交、确认 OK 但 health 故障、COMMIT_UNKNOWN 三类结果，已确认 free 不再复用。
- V-54：checkpoint/close 与 gate 等待者、data persist pin、已提交未 apply、WAL 满自退出交错；证明无遗漏快照、无自等待、无 lease 提前释放和故障后覆盖恢复证据。
- V-55：资源身份失证、cleanup token、未知 pfree 重开查询与多 arena 同时运行；证明旧 FD/地址不盲重试、清理责任不丢失，一个实例的 fault 不阻断另一个实例的正常路径。
