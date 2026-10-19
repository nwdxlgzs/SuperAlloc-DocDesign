# 运行、观测与故障处理指导

状态：未来运行手册，无已部署系统。具体status、结构及合法调用见[C ABI](c-abi-spec.md)，默认额度见[配置](configuration.md)。本手册描述调用方能采取的动作，不引入自动后台线程、回调或外部堆回退。

## 1. 建立可解释的观测

实例创建后保存UUID/incarnation、容量、来源、模式、P/G、defaults版本、求值额度、能力阶段及本地文件系统保证。每次采样记录时间、健康状态、统计一致性/validity；无效字段显示“不可用”，不填0。

| 量 | 解释 | 常见误读 |
| --- | --- | --- |
| reserved C | 独占虚拟地址长度，含全部内部区 | 当作RAM消耗或payload可用量 |
| broker free与最大连续候选 | 可用范围总量及连续性，还需符合backing边界 | 总free够就一定能固定地址映射 |
| requested/usable/live | 请求量、slot容量、尚未被owner处理的live | 无侧表却声称精确小对象requested |
| private/empty/retired | 独占span、空缓存、已逻辑释放待回收 | 把remote发布或retire当作即时OS释放 |
| metadata used/reserved | 已划入管理区的容量与正在预付的新容量 | 未用pool库存不算metadata，或每record重复收费 |
| backend charge / resident | 资源承诺与系统驻留采样 | 同一指标或必定同步下降 |
| fine/ordinary units | 项目逻辑映射单元计数 | 等同真实内核VMA/handle数 |
| quarantined | 不能确认可安全复用的区间 | 只是普通碎片，可直接塞回free树 |

低成本并发统计有采样误差和不同时间点；精确对象总量需要owner协作或quiescence。统计调用不能强抢私有heap；RT单次返回本次工作计数，不能为了全局精确计数在free中增加竞争原子或全池扫描。

## 2. 状态码到动作

| 状态 | 首先检查 | 合法后续动作 |
| --- | --- | --- |
| NO_ADDRESS_SPACE | 容量、位宽、连续VA、实际P | 创建前显式选择另一容量/配置；不把失败实例当有效 |
| OUT_OF_MEMORY | 当前总可用量、活对象、retired/private滞留 | 释放不再使用的对象，预算维护或应用背压；不重复free |
| NO_CONTIGUOUS_RANGE | 对齐、洞分布、最大区间、backend边界 | 改placement/拆分应用请求或在安全点回收整span；不移动活malloc对象 |
| METADATA_LIMIT | 1002、目录前缀、ctx/views、临时报告/恢复预算 | 减少需要管理的资源；维护可回收项；下次创建调大预算 |
| COMMIT_LIMIT | 1001、old/new双份峰值、retained section | 释放确认可释放的backing、调整事务范围；不能忽略峰值 |
| LIMIT | error.resource_key和used/reserved/limit | 处理对应ctx/view/Pending/root/目录上限；不等同物理内存不足 |
| WOULD_BLOCK | RT的本次扫描与准备容量 | owner在预定时点维护；应用背压/延迟；不向普通堆隐式补池 |
| BUFFER_TOO_SMALL | 输出required_count/结构尺寸 | 扩充调用者buffer并重新执行预检；旧estimate未保留资源 |
| STALE_CURSOR | 分页期间ledger变化，或revision已饱和 | 一般变化可从头重查；持续写入时安排同步快照。饱和后重启游标仍会失败，使用标量诊断并安排实例正常退出/重建，不能无限重试 |
| COLLISION / BUSY | 当前owner、目标Pending、roots、绑定及关闭前置 | 等相关操作完成或解除应用引用；不绕过句柄强制覆盖 |
| UNSUPPORTED / INCOMPATIBLE_* | 来源、模式、页几何、ABI/格式/能力阶段 | 使用声明支持的接口/环境；不自动重写或升级store |
| OS_ERROR / IO_ERROR | native错误、stage、已知无效果与清理输出 | 按明确结果重试或清理；不能仅凭通用错误推断无副作用 |
| PARTIAL_FAILURE / QUARANTINED | 本次完整范围report与健康状态 | 仅使用确定有效片段；保持隔离并维护/整体丢弃 |
| COMMIT_UNKNOWN / WRITE_FAULTED | 主文件、事务身份、尚未判定对象 | 停止store写入/复用，quiescent abort-close后恢复查询 |
| CORRUPT | 原始文件镜像/最小触发序列 | 保留证据、拒绝打开写入；通过备份或后续离线工具处理 |

错误优先级按执行阶段，不是全系统状态证明。metadata增长作为独立内部事务可能已发生，但失败的用户申请仍无对象；报告要区分这种合法管理变化与用户payload效果。

## 3. 维护调度

服务端可在请求批次末、游戏在帧边界、语言runtime在worker安全点调用thread_flush。arena_maintain处理shared heap、central/orphan/退休资源，每次传对象、页数、步骤三维总预算；span处理计入步骤和实际页数，单纯对象drain不消费页数。记录本次进展和remaining提示，再由应用决定下个调度点；不循环忙等休眠owner归还其全部缓存。

额度耗尽时先检查长期活对象、partial slab和retained backing；反复执行无进展maintenance不会制造连续空间。私有span到达上限时移交或共享路径不应让线程的总活对象量永久受4S限制。RT维护独立遵守固定K和owner，不让压力路径调用普通OS回收。

在持久模式，checkpoint会暂停该实例元数据修改并顺序扫描固定目录；N、日志大小与持久IO决定其成本。应用可以主动在低负载时checkpoint；WAL满时内部也必须先完成合法checkpoint才继续。没有实测停顿时间前不能把它放进需要固定墙钟SLA的路径。

## 4. 异常清理和重开

create/open/init失败通常不产生有效arena；若已取得OS资源无法完整释放，则返回PARTIAL_FAILURE和非空caller-owned cleanup token。token是单一责任凭据，不复制成两个清理者，不把其中字节当普通arena。按能力调用cleanup_retry直到已确认资源收口；不可判定的native close状态不能盲重试可能已复用的FD。详细两种token状态见ABI。

destroy/close/discard在进入清理后也可能把最后控制区责任移交token。只有BUSY等OS前拒绝明确保留原handle可用；得到责任移交结果后，旧arena句柄不能再作为普通入口使用。cleanup成功清空token，不重新销毁同一资源。借用buffer的物理映射一直归调用者；库清理不会替其unmap。

WRITE_FAULTED不是“稍后自动恢复”的状态。应用先终止业务对该store的访问，在abort-close完成后重新open；后者按有效checkpoint/WAL决定未知操作是否提交。其他arena可独立继续。close与文件删除/擦除是不同动作；首版没有在线repair、自动截断坏尾或格式迁移。

超时不等于取消：一个仍在OS或commit中的同步调用保有完成责任，不能由维护线程抢Pending、释放pin或重放同一步。已经获得OK的普通free不得再次调用；durable操作得到OK同时health=WRITE_FAULTED时，其成功事实也必须保留，停止的是后续写入。具体返回组合和局部故障边界见[失败与恢复](failure-recovery.md)。

checkpoint镜像第一半已确认、第二半明确无效果失败时，实例可以health=READY但admission被CHECKPOINT_HELD阻塞，新修改返回BUSY。调度一个显式checkpoint继续既有镜像，或quiescent close/abort-close；不要持续创建新业务请求等待它自动解除。结果不明则转WRITE_FAULTED，只走恢复流程。

若局部reservation的native所有权失证，不对同地址盲目fixed重建，也不以整arena unmap掩盖残留。只清理仍有确切所有权证明的资源；查询到“地址现在可用”本身不是原所有权证明。控制结构失信或无法界定的后端破坏须停止实例使用，不能把常规局部隔离能力宣传为任意内存破坏的自愈。

## 4.1 高负载下的恢复与背压

应用按实例分别调度维护和业务准入；先判断是活对象占用、远程归还待收、退休积压、连续碎片还是后端资源保留，再选择动作。业务到达率超过当前可持续服务率时限制并发/推迟新分配请求，让已有合法释放继续；错误重试同样计入应用的并发上限。不能通过不断增大扫描批量掩盖单次尾延迟，也不能为提高成功率取消既定配额。

轮询步长和采样间隔由应用SLA决定，库不增加隐藏计时器或后台线程。重复维护没有进展时记录阻塞原因，等待owner协作、后端可恢复事件或下一个调度点，不在请求线程持续忙等。已退出队列且正在执行的记录不能被第二个维护者重复领取；失败记录回队尾使其他可进展责任继续得到访问。

恢复后先确认未知事务的实际分配/root结果，再恢复业务准入。需要增加配额时按创建配置规划新实例；RT仍只使用预备池，不能把普通慢路径当作超负载补救。性能验收同时看积压是否消退、各线程是否持续有进展和p99/p99.9，不能只看短时间完成量峰值。

## 5. 数据与配置变更

正常重开前核对页大小、格式版本、主文件身份及应用payload版本；相对offset只解决地址变化，不解决C++布局或业务schema。备份必须在应用一致性点结合payload同步和allocator checkpoint，再复制完整文件；边写边复制的结果不自动具有一致性。

需要更大容量、目录或不同页几何时，创建新的store，由应用离线枚举/转换/重建引用与roots，验证后切换。不能直接修改superblock容量字段，不能盲目memcpy含裸指针的业务对象。首版库不提供通用业务迁移工具；流程与兼容承诺见[发布策略](release-policy.md)。

## 6. 缺陷报告资料

报告最小调用序列、涉及模式/容量/页几何、求值配置、线程归属与同步、status/native stage、版本和触发概率。并发问题附调度暂停点或seed；持久问题附经授权脱敏的故障镜像与中断边界；性能问题附原始run数据和[基准manifest](benchmarking.md)。不要用只剩一张平均吞吐截图代替可复现记录。

内部诊断默认不写文件、不格式化长字符串、不回调应用；上层选择如何记录。生产记录避免转储业务payload；元数据损坏与外部程序改写/越界属于不同根因，都应在证据中区分。
