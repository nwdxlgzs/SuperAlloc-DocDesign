# S-05：持久格式 v1 候选规格

状态：**设计完成候选，未实现、未执行平台 spike、未发布格式或 ABI**。本文把 [持久化契约](persistence.md) 收敛为可实现的字节布局与恢复算法。`v1` 是本候选的识别号；完成 I-10A、格式故障模型和交叉平台验证之前，不允许产生声称长期兼容的生产文件。后续修改候选必须更新本文及设计决策，不能把未验证结论标为格式冻结。

## 1. 编码、完整性与适用范围

- 所有整数使用无符号 little-endian；显式读写字段，不通过 `packed struct`、C++ 对象复制或 native ABI 序列化。
- 字节范围均为半开区间。所有长度相加、乘法、取整与转换在执行前检查；磁盘 u64 能编码不表示 32 位进程能寻址。
- 校验统一选择 **CRC-64/ECMA-182**：width=64，poly=`0x42F0E1EBA9EA3693`，init=0，refin=false，refout=false，xorout=0；ASCII `123456789` 的 check=`0x6C40DF5F0B497347`。存储 CRC 数值也用 LE。
- 对记录计算 CRC 时，仅将该记录自己的 CRC 字段视为全零，按文件中的字节顺序覆盖指定完整长度，包括要求为零的 padding。父记录可以覆盖已经填入 CRC 的子记录。
- CRC 用于检测撕裂和偶发损坏，不是密码学认证，不保证检测任意恶意构造或校验碰撞。解析器即使遇到碰巧通过 CRC 的输入也必须先验证范围、数量和状态。
- 首版只允许一个进程管理一个主文件；外部 session mappings 不进入本格式。主文件精确逻辑长度为 C，不隐式截断或扩容。
- 本候选支持 `CLEAN_REOPEN` 和 `RECOVERABLE_METADATA`；`VOLATILE` 没有本格式。页几何或分配几何不兼容时拒绝重开，不迁移 payload。

## 2. 固定区域与可增长运行时元数据

令 C 为 arena 总容量，P 为实际系统页大小，G 为 `max(64KiB, P, backend_allocation_granularity)`。本候选geometry=1明确限制 **P为4KiB或16KiB、G=64KiB**，后端allocation granularity必须是整除64KiB的二次幂；不满足返回UNSUPPORTED，不能改变文件几何。C为G整数倍，16MiB≤C≤4TiB，并满足当前ABI的PTRDIFF_MAX页对齐上限；64位4TiB低于其该上限，32位另受实际连续VA限制。重开要求P和G与文件相同，即使某次布局碰巧可跨页映射也不自动放宽。

默认区域大小按 [配额规格](configuration.md) 同一公式计算：

- `R = align_up(ceil(C/16), G)`：runtime reservation，包含 bootstrap、目录镜像、索引、线程及恢复工作区。
- `J = align_up(clamp(ceil(C/128), 256KiB, 64MiB), G)`：一个 WAL 区。
- `K = align_up(clamp(ceil(C/64), 256KiB, 64MiB), G)`：一个 checkpoint 区，两个共 `2K`。
- 默认 root 数 Q=64。root 配额是条目数，checkpoint 配额是双份合计 `2K`，journal 配额是 J。
- `N_cap = min(floor((K-4096-64Q)/64), floor((R-128KiB)/768))`。
- 默认 allocation slot 数 `N = floor(N_cap/1024)*1024`；结果小于 1024 时自动配置无效。显式 N 可以从 1 起，但仍必须通过完整大小与实际重建预算检查。

768 B 是默认 profile 的每 slot 预算信用，不是任意稀疏布局的充分性证明。实际分配和 open 仍按 [S-01 单价](metadata-budget.md) 计算 slab、radix、view、目录和工作区；不足明确拒绝。N 是目录身份上限，不是无条件可同时分配 N 个任意大小对象的承诺。

| 区域 | 文件 / arena-relative 起点 | 长度 | 后备与用途 |
|---|---:|---:|---|
| Superblock 0 | 0 | G | 主文件独立 view |
| Superblock 1 | G | G | 主文件独立 view |
| Runtime reservation | 2G | R | arena 内匿名 islands；同 offset 文件字节不作恢复输入 |
| WAL | 2G+R | J | 主文件独立 view，可按后端边界细分 |
| Checkpoint 0 | 2G+R+J | K | 主文件独立 view |
| Checkpoint 1 | 2G+R+J+K | K | 主文件独立 view |
| Payload | 2G+R+J+2K | C−前述起点 | 主文件按需 extent views，不整体建立单个不可拆大 view |

持久主文件对象的 payload offset 等于它的 arena-relative offset。运行时元数据只在 R 内增长，不能把新控制页插入 payload 并改变已发布对象地址；R 上限在创建时固定。**整个R自创建起属于METADATA ownership**，内部子态为RESERVED/ACTIVE/PENDING/QUARANTINED，不进入用户FREE。option1002在创建时按整个R地址预付为used=R；R内增长只是将未启用单元从RESERVED转ACTIVE，不重复收取metadata地址额度，实际backing、view和records另计。R可以很大而不全提交，控制结构仍按需增长。主文件对应R的逻辑字节保留，不分给持久对象，不要求写入匿名锁和线程状态。

如 R 被耗尽，返回 `METADATA_LIMIT`；日志空间不足可以先 checkpoint，checkpoint 无法完成时返回明确错误；N 个目录槽耗尽返回 `LIMIT`，不能靠覆盖旧 FREE 的 generation 信息伪装无限身份空间。扩大 R/J/K/N 或修改区域位置属于离线格式迁移，不是运行时 set-limits。

显式配置接受上界：`1≤N≤UINT32_MAX`、`0≤Q≤UINT32_MAX`；R/J/K均为G整数倍，J≥G，K≥`alignG(4096+64Q+64N)`，R≥下节M_init，固定区总和F<C且C−F≥G。每项字节值均小于C，所有表达式先验证无溢出。J/K的64MiB只是默认公式封顶，显式值可以增加到上述有限容量允许的值；没有超越C的磁盘metadata扩展文件。大N/Q还必须通过R内实际工作区/记录与当前ABI数组长度检查，不能只通过u32编码检查。

### 2.1 Runtime目录镜像、辅助区和scratch

下列布局位于R起点内，均不是持久格式输入。各段从G边界开始：

| 段 | 长度 | 内容 |
|---|---:|---|
| Bootstrap | G | S-01控制块、初始记录库存、单独radix根与应急库存 |
| 持久控制段 | `U=alignG(512+4096+512+64Q)` | control512B，stream scratch4096B，WAL/after-image scratch512B，root镜像64Q B，尾部零初始化 |
| allocation镜像 | `A=alignG(64N)` | 每槽一份完整已验证64B目录记录；未提交事务不覆盖已提交镜像 |
| slot辅助段 | `X=alignG(64N)` | 下表SoA数组，恢复后继续作为身份索引和查询辅助，不再分配第二份 |
| 后续metadata单元 | `R−M_init`，`M_init=G+U+A+X` | RESERVED单元按需转ACTIVE；span/radix/位图/ctx等pool从这里或bootstrap未用空间建立 |

control512B固定分账：prefix16；commit/admission两个64B LockSlot在16/80；16个u64计数在144起依次为wal_epoch、next_lsn、last_committed_lsn、super_generation、checkpoint_generation、checkpoint_lsn、open_session_generation、active_operations、next_wal_slot、free_slot_count、live_count、retired_count、runtime_revision、durable_op_id、root_count、cleanup_pending_count；8个u64区内offset在272起依次为allocation镜像、root镜像、辅助段、stream scratch、WAL scratch、当前CP映射、WAL映射、backend文件控制状态；8个u32状态在336起为admission_open、dirty_state、health、active_checkpoint、mode、crc_kind=1、recovery_phase、reserved=0。字段末端368，其余144B零保留。它们只供runtime使用、不原样序列化；锁槽遵循S-01平台验证。

slot辅助采用列式数组，每列连续；u64列起点按8B对齐，u32列起点按4B对齐。N为奇数时两列4N合并后使下一u64列仍在8B边界，无需列间padding。总长恰为64N：

| 列起点（相对辅助段） | 元素 / 数量 | 用途 |
|---:|---|---|
| 0 | u32×N | order_by_span，heapsort的目录下标排列，0..N−1 |
| 4N | u32×N | free_id_stack，只保存FREE的可复用allocation_id，LIVE/RETIRED不得入栈 |
| 8N | u64×N | 对应runtime span descriptor的区内offset，0表示无；不是持久指针 |
| 16N | u64×N | 对应runtime extent descriptor的区内offset |
| 24N | u64×N | 本slot最后已应用LSN，checkpoint初始统一为其截点 |
| 32N | u32×N | direct_root_refcount，重建时清零并由Q根累加，最大Q≤UINT32_MAX |
| 36N | u32×N | 解析/重建状态位；仅定义VALID、LIVE、RETIRED三位，其余为零 |
| 40N | u64×N | 同span下一个allocation_id，0为空，便于受界枚举 |
| 48N | u64×N | 已检查的payload_end，避免重复不安全算术 |
| 56N | u64×N | 查询revision/恢复临时标记，只在控制锁或quiescent阶段读写 |

所有辅助u64是受慢路径锁保护的普通值，不要求ARMv7的u64 lock-free。恢复排序只原位改order列，最坏O(N log N)比较、常数栈深度；扫描WAL只需一个256B槽和64B after-image，包含在512B scratch中。CRC流式覆盖super/checkpoint时每次至多4096B，包含页大小大于4096的文件也使用普通有缓冲文件I/O，不使用要求更大对齐的DIRECT_IO。写checkpoint可由镜像和零scratch流式生成，不需要第二个K缓冲或另一个64N目录。

恢复span及其位图是最终runtime记录，不是免费scratch：由排序后的连续同span组建立，分配前精确预估数量和pool取整。恢复前不需要读取整个payload，也不需要每个对象保留单独WAL历史；固定目录、辅助区、scratch及后续pool共同构成完整内存上界。

### 2.2 Create与open的options覆盖规则

create按defaults profile1与显式配置产生几何。open不重新套公式覆盖文件；下表是全部option族的处理，未列出的键拒绝：

| 输入 | open省略 | open显式值 |
|---|---|---|
| cfg.mode | 必填并匹配主文件 | 不允许模式转换 |
| cfg.capacity | 0采用文件C | 非零必须等于C |
| cfg.defaults_version | 必须1 | 未支持版本拒绝 |
| cfg.flags | 0 | 只允许TRACK_REQUESTED/DIAGNOSTIC_CHECKS；按runtime开销预检，持久requested本来即精确保存 |
| 1002 metadata、1010 roots、1011 journal、1012双CP、1106目录槽 | 分别取R/Q/J/2K/N | 必须与文件完全相等；包括显式0也不能改写几何 |
| 1001 backend、1003 ctx、1004 private span、1005 empty span、1006 fine views、1007 ordinary views、1008 pending | 按profile1及文件C求运行默认 | 允许覆盖，但不得低于重建实际需求；每项范围/零值规则遵循configuration |
| 1009 RT payload | 0 | 仅0；持久模式不提供RT |
| 1103 fine views per map | 默认4096 | 允许配置，仍受arena view/metadata额度 |
| 1101 drain、1102 RT scan、1104/1105 per-ctx span | profile1默认 | 接受标准合法值并可查询；不为持久对象引入普通ctx slab或RT能力，不改变文件布局 |

lower backend/ordinary-view/ctx等运行配额可使曾成功创建的文件无法打开，应在handle发布前返回对应资源错误。metadata额度不能比R低，降低“实际启用量”通过后端预算而不是伪报更小1002。调用方必须保持输入file_ref有效到open返回，早期头读取不是可信模式切换依据。

1004/1005与1101/1102/1104/1105在持久前端属于不产生普通ctx slab或RT资源的保留运行策略；标准合法值可配置和查询，不因open/create入口不同而改变接受规则。首版持久pool的物理回收仍按自己的事务和backend规则执行，不借这些闲置额度偷偷扩大N/R。

### 2.3 16MiB 的确切默认布局

下表取 G=64KiB，适用于通过该粒度后端验证的 4KiB/16KiB 页配置。

| 区域 | 起点 B | 长度 B |
|---|---:|---:|
| SB0 | 0 | 65,536 |
| SB1 | 65,536 | 65,536 |
| Runtime | 131,072 | 1,048,576 |
| WAL | 1,179,648 | 262,144 |
| CP0 | 1,441,792 | 262,144 |
| CP1 | 1,703,936 | 262,144 |
| Payload | 1,966,080 | **14,811,136 = 14.125MiB** |

N=1024，Q=64。每份 checkpoint 的已定义内容为 `4096 + 64×64 + 1024×64 = 73,728B`，其余 188,416B 为零填充及未来离线配置空间，不自动增加公开 allocation slot 配额。J 容纳 1024 个 256B 事务槽。

16MiB 持久初始化不沿用“只提交一个 64KiB 控制岛”的 volatile 承诺。Q64时控制段原始9216B取为G，N1024时A=X=G，因此M_init=4G=256KiB。R中其余12个G仍是METADATA/RESERVED，尚未提交；1002的used仍为整个R=1MiB，不能只报M_init。文件metadata views的backend资源另计。

存在一个可闭合的全N默认对象工作集：payload最多226个64KiB单元；小对象使用64KiB slab，大对象后端长度至少按G取整。64位/4KiB页保守账为：M_init=262,144B；226个单元各 `(Slab192+Extent144+Backend112+PageTile160+最大bitmap512)=1120B`，共253,120B；最多227个额外FREE描述符32,688B；另留G=65,536B给pool头、取整、固定文件views与runtime增长单元描述符。合计613,488B，可安排在10G=655,360B内，小于R=1MiB。

没有漏计radix：16MiB/64KiB tile只有256项，D=1，唯一2080B根已在bootstrap库存内；226个PageTile已逐单元计入。Slab192比Large96更大，按Slab计所有单元是保守包络；跨多个G的大对象实际只需一个descriptor，因此该按226单元上界仍不低估此项。bitmap每slab至多512B已计入，与64N身份索引不是同一资源。

pool剩余分账规则：后续单元允许在同一G里安排多个family chunk，不按每个family再收费一个G。上述额外G覆盖至多16family的64B头和至多63B对齐空洞（≤2032B）、10个runtime单元每个最多两条Extent144和一条Backend112（≤4000B）、固定SB/WAL/CP及其分割节点（本例按16组Extent144+Backend112计≤4096B）和必要RangeTxn/TxnStep；现有bootstrap库存可以抵扣但本例不依赖该抵扣。其余余量足够上界内的整体G取整。实施必须核对真实后端没有要求额外未计单元；出现更大native结构须重算后拒绝或修订候选，不能默默花超R。

该账针对无额外 session mappings、无额外线程增长、P≤16KiB 的最小持久 profile，证明候选没有在16MiB时天然超预算；它不是任意负载的最坏情况证明。实现必须验证真实结构、位图和混合family分块实际落入预留分账；session/线程资源仍单独预检，不能花掉恢复所需额度。

| C / G=64KiB | R | J | K（单份） | 默认 N | WAL 槽 |
|---|---:|---:|---:|---:|---:|
| 16MiB | 1MiB | 256KiB | 256KiB | 1,024 | 1,024 |
| 64MiB | 4MiB | 512KiB | 1MiB | 5,120 | 2,048 |
| 4GiB | 256MiB | 32MiB | 64MiB | 349,184 | 131,072 |
| 4TiB | 256GiB | 64MiB | 64MiB | 1,047,552 | 262,144 |

4TiB 仍受约一百万目录槽的默认上限，绝非可存任意数量的16B持久对象。J/K 的64MiB默认封顶不是字段编码的无限扩展承诺；显式配置还必须符合实现接受的资源上界与解析预算。

## 3. Superblock：两个 G 字节槽

所有偏移相对槽起点。固定头512B，512至G末尾为零。CRC覆盖完整G字节，CRC字段@504按零计算。

| 偏移 | 类型 / 长度 | 字段与约束 |
|---:|---|---|
| 0 | bytes[8] | magic=`SALOCV1\0` |
| 8 / 10 | u16 / u16 | major=1、minor=0 |
| 12 | u32 | header_bytes=512 |
| 16 | u64 | required_features=0；未知非零位拒绝 |
| 24 | bytes[16] | 非零 arena UUID |
| 40 | u64 | super_generation，从1递增、不回绕 |
| 48 / 52 | u32 / u32 | mode：1=CLEAN_REOPEN、2=RECOVERABLE_METADATA；state：1=CLEAN、2=DIRTY |
| 56 / 60 | u32 / u32 | G、P |
| 64 / 72 | u64 / u64 | arena_capacity、file_bytes；首版都等于C |
| 80 / 88 | u64 / u64 | runtime_offset、runtime_length=R |
| 96 / 104 | u64 / u64 | journal_offset、journal_length=J |
| 112 / 120 / 128 | u64×3 | cp0_offset、cp1_offset、checkpoint_length=K |
| 136 / 144 | u64 / u64 | payload_offset、payload_length |
| 152 / 156 | u32 / u32 | allocation_slots=N、root_slots=Q |
| 160 / 164 | u32 / u32 | active_checkpoint=0或1；reserved=0 |
| 168 / 176 | u64 / u64 | checkpoint_generation、checkpoint_lsn |
| 184 / 192 | u64 / u64 | wal_epoch、next_lsn=checkpoint_lsn+1 |
| 200 | u64 | open_session_generation；每次可写打开递增 |
| 208 / 212 | u32 / u32 | allocation_geometry=1、durability_contract_id |
| 216 | u64 | optional_features=0；首版任何未知非零值拒绝 |
| 224 | u64 | 所指checkpoint的CRC副本 |
| 232 | u64 | root_geometry=1 |
| 240–503 | bytes[264] | 必须为零 |
| 504 | u64 | super_crc64 |
| 512–G−1 | bytes | 必须为零 |

durability_contract_id 首版仅允许1，固定要求至少达到公共API的`FILE_BARRIER=3`，即已完成经平台验证的文件内容屏障。平台可执行更强屏障并报告实际阶段，不能降级；未知profile拒绝。`DEVICE_BARRIER=4`只在已验证能力支持时用于显式persist，首版不提供另一个含糊的store默认。该字段不能证明硬件兑现合同；新环境无法满足profile1则拒绝可写打开。

API只接受调用方提供的file_ref/FD，没有path入口。新路径创建与目录项持久性由调用方完成；库对要求自身保证namespace durability的请求返回UNSUPPORTED。本格式不将文件内容flush冒充路径已持久化。

两个槽可以在更新中暂时具有不同 generation。选择最高**完整有效**候选，但必须验证其引用的checkpoint与geometry，不能把“最高头CRC正确”当作全部有效。相同generation但不同内容视为损坏。已经完成可写发布后，两槽应指向同一checkpoint/epoch/state；generation可以相邻不同。

## 4. Checkpoint：双固定区

每份K字节：4096B header，Q条64B root记录，N条64B allocation记录，尾部零填充。位置由公式直接计算，不接受记录内任意自选位置。CRC覆盖完整K字节，包括各子记录CRC与零填充，只有header自己的CRC字段@104按零处理。

| Header偏移 | 类型 | 字段 |
|---:|---|---|
| 0 | bytes[8] | `SACKPT1\0` |
| 8 / 10 / 12 | u16/u16/u32 | major=1、minor=0、header_bytes=4096 |
| 16 / 24 | u64/u64 | checkpoint_generation、checkpoint_lsn |
| 32 | bytes[16] | arena UUID，必须与super一致 |
| 48 / 52 | u32/u32 | N、Q |
| 56 / 64 | u64/u64 | allocation_offset=`4096+64Q`；root_offset=4096 |
| 72 | u64 | used_bytes=`4096+64Q+64N`，不大于K |
| 80 / 88 | u64/u64 | live_allocation_count、next_lsn=checkpoint_lsn+1 |
| 96 | u64 | allocation_geometry=1 |
| 104 | u64 | checkpoint_crc64 |
| 112 | u64 | retired_slot_count |
| 120–4095 | bytes | 必须为零 |

### 4.1 Allocation 记录：固定64B，FREE也保留代次

第i条的allocation_id必须等于i+1，i从0起。不存在链式文件指针或自由长度记录。

| 偏移 | 类型 | 字段 |
|---:|---|---|
| 0 / 8 | u64/u64 | allocation_id、generation |
| 16 / 24 | u64/u64 | payload_offset、requested_length |
| 32 / 40 | u64/u64 | span_offset、span_length |
| 48 / 50 / 51 | u16/u8/u8 | class_id、alignment_log2、state |
| 52 | u32 | flags=0 |
| 56 | u64 | record_crc64，覆盖64B |

state：0=FREE、1=LIVE、2=RETIRED。初始FREE generation=0；每次分配将FREE代次加1。free保持最后generation但清零位置、大小、class、alignment和flags；generation已达u64最大值的slot释放后进入RETIRED，永久不再分配。此规则避免恢复后把旧handle误认成新对象。任何代次、LSN、epoch或super代次溢出都返回LIMIT，不回绕。

LIVE requested_length>0，`4≤alignment_log2<64`且所表示对齐不能超过C；必须先验证log2再执行shift。payload范围在payload区域内。class_id为1..44时使用普通44档size class的几何v1：1..16对应`16×class_id`；17..44令`k=floor((class_id−17)/4)`、`r=(class_id−17)%4`、`B=256×2^k`，stride依r为`5B/4、3B/2、7B/4、2B`。span_length=64KiB、span_offset按G对齐，slot offset相对span为stride整数倍，requested≤stride且payload_offset满足记录对齐。不能仅凭class足够大推断每个slot都满足更高对齐；不能满足的请求转独立大对象。

class_id=`0xffff`表示独立大对象/特殊对齐：span起点与长度按G对齐，payload_offset满足请求对齐且`offset+requested<=span_end`，usable长度为span_end−payload_offset；span前部对齐padding仍归该对象。小类usable为stride。所有span占用必须与其他span不重叠；同一slab的多条LIVE记录必须具有一致class/span且不同slot。持久FREE账本由LIVE spans及固定控制区域的补集推导，不单独持久化可冲突的第二棵空闲树。

小对象恢复从LIVE记录重建runtime slot位图，不依赖文件payload中的freelist。全部对象释放后的空slab不必继续存在于持久账本，缓存保留只是当前session状态。一个仍有LIVE对象的slab整段仍属于该主文件后端，不能将其中空slot或空页交给外部file map。

每slab合法slot数为`floor(65536/stride)`，payload记录的slot_index必须小于该数，尾部不足一个slot的字节不可分配。runtime bitmap长`ceil(slot_count/8)`，最大512B；初始全部清零，按LIVE记录逐个置位，遇已置位为CORRUPT，最后一个字节超出slot_count的高位保持零。该bitmap不落盘，无独立持久CRC，其来源是已经通过CRC和状态检查的目录；恢复时不能信任R或payload遗留bitmap。

每个64B allocation记录必须校验CRC。FREE只允许ID/generation/state/CRC非零，且generation小于UINT64_MAX；RETIRED除这几项外同样为零且generation必须为UINT64_MAX；LIVE的flags=0，几何和范围全部通过。所有记录下标按N上界检查，不能以记录内ID去申请新数组。checkpoint header的live/retired计数需与重放前目录逐项计数相等，WAL重放后重新计算当前计数。

### 4.2 Root 记录：固定64B

| 偏移 | 类型 | 字段 |
|---:|---|---|
| 0 | u64 | root_id，等于记录下标，范围0..Q−1 |
| 8 / 16 | u64/u64 | allocation_id、generation |
| 24 | u64 | byte_delta=0，root只能指对象起点 |
| 32 | u64 | last_update_lsn，初始0 |
| 40 / 44 | u32/u32 | flags=0；state：0=EMPTY、1=BOUND |
| 48 | u64 | record_crc64，覆盖64B |
| 56 | u64 | reserved=0 |

EMPTY的allocation_id、generation、delta为0。BOUND必须解析到本arena当前LIVE且代次匹配的allocation。清空root可以保留其last_update_lsn，表示这次清空已进入提交顺序。不存在磁盘native pointer或native handle。

公开C ABI的phandle/ref含UUID等便利字段，不能按其内存布局直接写成上述记录。普通内部引用可以含byte_delta，但应用需自己的LE序列化协议；本格式root严格delta=0。

## 5. WAL：固定256B事务槽

J是256的整数倍；slot_count=J/256。没有变长记录、跨槽事务或环形覆盖。每个元数据事务占一个槽：前128B redo、后两个完全相同的64B commit副本。实例内按持久提交gate串行，首版无group commit。

单个事务仅为ALLOC、FREE或ROOT_SET之一。大对象分配仍只改变一个allocation目录槽；小对象所在slab归属可由记录推导，不增加隐含第二事务。外部filemap不写WAL。

### 5.1 Redo 128B

| 偏移 | 类型 | 字段 |
|---:|---|---|
| 0 | bytes[8] | `SAREDO1\0` |
| 8 / 10 / 12 | u16/u16/u32 | version=1；type：1=ALLOC、2=FREE、3=ROOT_SET；record_bytes=128 |
| 16 / 24 / 32 | u64×3 | wal_epoch、transaction_id、lsn；transaction_id=lsn |
| 40 | u64 | slot_index，从0连续递增 |
| 48 / 52 | u32/u32 | payload_bytes=64、flags=0 |
| 56 | u64 | redo_crc64，覆盖128B |
| 64 | bytes[64] | 目标allocation或root记录的完整after-image，含其自身CRC |

ALLOC要求旧slot为FREE、after代次恰为old+1且after为LIVE；FREE要求当前LIVE与after的ID/代次相同，after为FREE/RETIRED；ROOT_SET验证当前对象与代次，直接指向的对象不能同时待free。root引用检查、选择slot及状态验证必须处于同一持久事务序列，不能在旧快照上各自通过。

### 5.2 Commit副本：各64B

| 偏移 | 类型 | 字段 |
|---:|---|---|
| 0 | bytes[8] | `SACMIT1\0` |
| 8 / 16 / 24 | u64×3 | wal_epoch、transaction_id、lsn |
| 32 | u64 | slot_index |
| 40 | u64 | 对应redo_crc64 |
| 48 | u64 | previous_committed_lsn |
| 56 | u64 | commit_crc64，覆盖64B |

两个副本位于事务槽+128和+192。它们不是硬件原子写；双份用于识别单份撕裂。至少一个完整有效副本可以证明该事务具有commit候选；两个都有效但不同则CORRUPT。有效commit必须匹配完整redo与after-image，不能只匹配LSN。

当前epoch第i个槽必须有`lsn=checkpoint_lsn+i+1`、`previous_committed_lsn=lsn−1`，交易从槽0连续追加。WAL不另设会在checkpoint之前被覆盖的唯一header；当前epoch、起始LSN和容量都来自superblock。

## 6. 提交、checkpoint与双super发布

### 6.1 RECOVERABLE_METADATA 单操作

1. 进入durable admission并预取runtime记录和配额ticket，此时不发布最终业务Pending。取得durable commit gate后短持broker，重新核对对象/root、待free状态、persist控制pin和目标范围，再建立Pending并释放broker。只允许gate→broker顺序，不能在broker/heap锁内等待gate。
2. 在control已有admission LockSlot下重查health并取得有限redo阶段许可，锁外在下一槽写完整redo，执行配置的文件内容屏障。
3. 再次重查并取得本事务commit许可；一次许可涵盖两个相同commit副本的写入及其完整commit屏障。屏障成功前不返回成功、不复用待free对象，不往其payload写freelist、不discard backing。许可先于并发fault时，有限已发阶段仍可完成；下一事务不能沿用许可。
4. 仍持gate并保留Pending及全部既有pin，在不持broker时短取metadata_pool结算已有ticket的实际/未知资源，解pool后才短持broker无失败地更新runtime权威目录、发布并撤Pending。Pending名额自身的reserved保守留至撤Pending后再退，其他资源先结算再发布；pin最后释放或移交，责任收口后才退出gate/admission并成功返回。无需新增结算状态或pin字段。gate内不得申请新资源，pool不反向等gate，也不与broker嵌套。文件I/O期间不持broker/heap锁。调用方收到成功前崩溃，事务仍可能已提交，恢复枚举负责暴露这种对象。
5. 确定无修改的预检失败可以正常返回；仅redo且从未获commit许可、可证明无commit的事务保持确定未提交结果。commit已可能生效而无法判定时返回COMMIT_UNKNOWN并进入WRITE_FAULTED。并发主文件persist使health故障时，本事务已确认commit且runtime发布仍返回OK；未启动commit的调用返回WRITE_FAULTED或自身已知IO_ERROR。故障后不签新写入、复用或破坏性回收许可，既有已发阶段按事实收口。

执行下一事务前必须完成上一事务runtime发布与ticket/pin收口。checkpoint的admission gate停止新调用并等到**全部已进入调用及其broker Pending/runtime apply/资源责任收口完成**，包括主文件data persist和commit gate等待者，不能只观察journal没有写线程。data persist以既有descriptor pin阻止并发pfree，后者最终验证遇冲突返回BUSY。需要建立新payload view的palloc在gate内最终验证/Pending之后、WAL之前按独立许可完成backend建立；失败不产生已提交allocation。metadata-growth不得在gate内临时执行，预计划不足须退出重新预取。health与阶段许可同步细节见[故障恢复](failure-recovery.md)，不改变本候选磁盘字段。

### 6.2 Checkpoint与WAL换代

WAL剩余槽为0时必须先checkpoint，不能绕回覆盖。已经admitted的请求若在gate内发现日志不足，必须在业务Pending/WAL写入之前释放gate、退出自身admission并归还ticket，再请求独立checkpoint协调者；不能等待包含自己的active数归零。其他已进入请求完成或同样无修改退出，checkpoint成功后原请求重新预检，不保留过期地址/配额报价。完整锁序见[操作协议](operation-protocols.md)。主动checkpoint使用相同流程：

1. 关闭durable admission并排空全部在途调用、data persist pin及资源收口责任；等待不持control/broker/commit锁，排空后重查health。session mapping不在持久目录中，但其runtime范围仍不可被snapshot过程错误分给对象。
2. 从已提交runtime状态构造inactive CP：复制全部N条（含FREE/RETIRED代次）与Q个root，填头及零尾，生成CRC；写入并同步。
3. 产生新manifest：checkpoint_generation+1，active CP切换，checkpoint_lsn=last_committed，wal_epoch+1，next_lsn=checkpoint_lsn+1；正常运行state仍DIRTY。
4. 写较旧super槽，使用新的super_generation并同步。
5. 将另一super槽写为同一manifest、再递增super_generation并同步。
6. **只有两次发布均确认后**才允许新epoch从WAL槽0覆盖旧槽，才可以在下一checkpoint重用旧CP区域，然后重新开放admission。

第二次镜像不是可省略的优化：如果仅发布一个新super就重用日志，后续回退到另一个旧super可能引用已被覆盖的旧WAL。正常可写状态中的两份super必须指向同一checkpoint/epoch/state。任何发布结果不明都停止写入；恢复可使用最新有效checkpoint，但不能在本次会话假设镜像已经成功。

若第4步已经确认、第5步后端失败能证明该发布完全无效果，原调用返回IO_ERROR，health仍READY，但内部进入CHECKPOINT_HELD，admission_open=0，新修改返回BUSY。持久control既有recovery_phase、dirty_state与当前manifest相关计数保留已完成的阶段；不扩展control512B或磁盘布局，不退回旧epoch、不复用旧WAL/CP。sa_checkpoint或符合quiescence的正常close可在admission锁下唯一claim并只重试尚缺的镜像；有executor则BUSY，不重写已确认第一槽，不创建新checkpoint，不无限内联重试。镜像确认后才开放admission；close继续所需的CLEAN发布/拆卸流程。重试结果未知转COMMIT_UNKNOWN/WRITE_FAULTED，abort-close可放弃本次会话而保留磁盘事实。缓存read-back不能证明故障写入未持久化；无法证明第5步完全无效果时直接进入未知结果合同。create/open尚未输出handle时按初始化失败与cleanup合同处理，不发布可分配的半成品实例。

epoch、LSN、checkpoint_generation与super_generation均为u64；一次协议需要的全部递增在写入之前检查（例如双super需至少两个剩余代次），不足时拒绝该写操作，安排离线迁移。checkpoint复制使用R内固定4096B stream scratch，单槽事务另用512B scratch，不额外申请一个K大小外部缓冲。

## 7. 创建、clean/dirty与重开

### 创建

调用方以不覆盖语义创建新空文件，传入明确授权初始化的file_ref，并负责其路径/目录项持久性。库先建立arena匿名bootstrap，再取得自己的文件引用、文件身份登记和owner lease，重查长度必须为0；任何非空文件（包括坏store和有效store）都拒绝格式化。设置逻辑长度C并完成文件内容屏障；构造CP0的全FREE目录、空roots、generation=1/LSN=0，CP1初始化为无效零区。WAL可以是稀疏零区。

发布指向CP0、epoch=1的双份CLEAN super并同步。返回可写arena前，再用双super发布DIRTY与递增session_generation。发生任何不确定错误都不发布可用handle，按操作协议保留可诊断的创建结果和必要cleanup责任；不删除调用方文件、不自动截断重试。失败后已变成非空的文件只能通过显式离线流程处理。

### CLEAN_REOPEN

可写打开只接受最高有效状态为CLEAN；在修改payload或运行时逻辑分配状态之前，以双super发布DIRTY。运行时alloc/free/root递增逻辑LSN并更新目录，不写WAL。正常close需要调用方先unmap全部session资源、detach并quiesce，再写inactive checkpoint并用上述双super协议发布CLEAN。此后才释放lease。

DIRTY clean store一律拒绝正常打开，不能自动恢复旧CLEAN checkpoint。发布DIRTY也必须两槽确认后才允许写入，防单槽损坏后误回退到旧CLEAN。CLEAN模式的payload persist不改变这一边界。

### RECOVERABLE_METADATA

CLEAN和DIRTY都可进入恢复；完整验证和任何写入前必须获取owner lease。为先建立严格区内bootstrap，open允许对调用方原FD执行有界只读探测：geometry1的G固定64KiB，在offset 0和65536各读至多512B头部到有界栈缓冲，只选取经数值上界检查的候选C/R；此时不信任其CRC/身份、不写文件、不取得库拥有的文件资源。建立匿名bootstrap后dup引用、取得登记和lease，再重读完整super/checkpoint；身份、长度或geometry与早期候选变化时撤销初始化并失败。

在handle发布前完成第8节scanner、runtime重建、一轮checkpoint换epoch及双super DIRTY发布。干净关闭以checkpoint发布CLEAN；WRITE_FAULTED仅允许诊断与abort-close，不声称已clean提交。调用方原FD必须在API调用期间持续有效。

## 8. 有界恢复scanner与拒绝条件

恢复在改写任何主文件字节之前完成只读验证与所需runtime预算预估：

1. 从两个固定槽读取G字节；验证magic、版本、CRC、所有reserved、文件精确长度和全部区域公式。未知features、非法G/P、32位不可寻址、重叠区域直接拒绝。
2. 按generation选最新物理完整候选并验证其CP完整CRC、子记录CRC与公式。两份manifest不同表示发布中断；只有能验证完整引用链的候选可用。不存在有效链返回CORRUPT。CRC完整链中的语义矛盾（例如交叠span、非法root、非法状态转移）必须CORRUPT，不能偷偷丢弃最新完整状态回退较旧逻辑数据。
3. 将N个目录槽与Q个roots读入R内镜像，验证ID和状态；未分配slot仍保留generation，不能跳过其校验。
4. 在RECOVERABLE模式按固定256B槽扫描整个J，最多J/256次。只忽略较旧epoch的陈旧槽；出现CRC有效但epoch大于super所声明值的commit返回CORRUPT，不以“不是当前epoch”掩盖日志已被覆盖。当前epoch从连续前缀逐条验证commit、redo、LSN、after-image与合法转移。
5. 遇到没有有效当前commit的第一槽记作tail边界，仍以有界扫描检查后续槽；若后面存在有效当前epoch commit则有中间断裂，返回CORRUPT。有效commit引用损坏redo、矛盾副本、非法状态转移都返回CORRUPT，不能当尾部忽略。
6. 尾部只有未提交redo、部分commit或旧epoch残留时，可忽略该未完成事务。该规则的故障模型是未确认写入可能撕裂/乱序、确认屏障之后介质履行持久合同；已确认数据之后的任意介质丢失可能无法与未提交尾部完全区分，不宣称CRC能解决这一信息缺失。
7. 由最终LIVE记录按span重建账本和slot位图，验证不同span无交叠、同slab无重复slot、roots全部指向当前LIVE代次。FREE补集不能包含控制区。恢复时不重建外部session文件handle。
8. 按真实记录、span与radix前缀计算runtime预算，超出R或配置资源上限返回METADATA_LIMIT，不先发布半初始化arena。所有数组只从已验证的N/Q/G计算，任何超限都不继续解析。
9. 写新checkpoint、换epoch、双super DIRTY发布后才开放写入口。若发布失败COMMIT_UNKNOWN，保持失败，不把该arena当作可写。

解析上界：头扫描2G，checkpoint至多2K（验证候选），日志J，allocation检查N次，root检查Q次；跨度去重/交叠检查采用R内有预算的排序索引，不能使用无界递归或按恶意offset分配任意内存。首版选择迭代原位heapsort的索引数组，元素为u32目录下标，需4N字节并计入slot辅助区；排序比较使用已验证u64范围，不用不安全指针减法。root直接ID索引，不扫描整个payload。

所有super/checkpoint未用字节逐块检查为零；旧epoch WAL例外，它保留历史槽且没有零填充要求。扫描器只读取槽内固定128/64/64段，不相信损坏record_bytes/payload_bytes触发变长读取。两个CRC有效super的UUID、C/P/G/R/J/K/N/Q和mode等不可变几何若矛盾，直接CORRUPT；不能挑一个更方便的版本打开。尾部长度不足一个固定记录、文件长度变化或读短均视为I/O/损坏，不把缺少字节补零伪造完整记录。

## 9. 结果、故障矩阵与迁移

- 记录slot已提交ALLOC而应用未接收handle：枚举可见LIVE，可能没有root；不自动GC。
- FREE在commit前失败：已确认无commit时旧对象仍live且payload未被改写；结果未知时由重新恢复后的ID/代次查询判断。
- ROOT_SET与FREE按同一事务序列验证；直接被root引用的对象pfree返回BUSY。间接引用由应用协议负责。
- data persist先于root发布；根事务不自动给用户数据附加事务、CRC或跨文件保证。
- 单个坏super可以回退到另一完整链；两个都坏、有效commit与redo矛盾、span交叠、非法root、未知几何全部fail closed。没有静默格式修复或“尽可能分配”模式。
- 版本、字段宽度、CRC、区域顺序、class表、代次规则改变需要新格式/geometry版本。首版无在线迁移，离线迁移先验证源、写独立目标、应用校验、完成目标屏障，再由应用替换路径；源文件在成功之前保持可用。

实现前必须执行 [平台验证剧本](platform-validation.md) 中PV-07～PV-10及[验证矩阵](validation.md)的持久场景：每个redo、commit副本、双super、checkpoint写入/屏障边界分别注入中断；模拟整页/扇区/字节撕裂和不违反已确认屏障的乱序；验证容量槽耗尽、generation上界与N/Q边界。文档推导通过不代表这些测试已执行。

本轮仅完成字节算术、CRC标准check向量和相对链接核对：`123456789`得到`0x6C40DF5F0B497347`；16MiB的M_init=262144B、checkpoint内容73728B、payload14811136B、示例runtime总账613488B=向上10G；64N辅助分组与control字段末端368B闭合。没有创建格式实现、二进制store、平台映射实验或断电测试。
