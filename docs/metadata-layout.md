# S-01：运行时元数据布局

状态：**文档推导完成，未编译、未实测**。本文件给出 runtime layout v0.1 的字段、偏移、槽位和生命周期；[容量预算](metadata-budget.md)据此计算初始自举和增长成本。它不是公开 C ABI，也不是持久文件格式。实际编译器布局、原子尺寸和 OS 锁适配由 I-03/I-04 验证。

## 1. 布局模型与适用条件

- W 为原生指针/uintptr_t 宽度，64 位取 8，Android32 取 4；LP64 与 Windows LLP64 均不使用 native long 表示固定宽字段。
- u8/u16/u32/u64 分别为 1/2/4/8 B。u64 在两种模型中都要求 **8B 对齐**；32 位实现必须显式满足，不假设编译器默认一致。
- ptr 表示原生宽区内指针、native handle 或不可重复 token；含义由字段定义区分。持久记录不复制这些字段。
- a32 为 4B 对齐的原子 u32；aptr 为 W 对齐的原子原生指针。要求 size/alignment 与原生整数或指针一致并通过 lock-free 验证，不满足即不能沿用本预算或 RT 能力。
- 普通记录至少按16B对齐；ArenaHeader、ThreadCtx、Heap、RTPool和SlabDesc按64B对齐；SlabDesc还明确按64B分区，其family的record_alignment=64。初始chunk头已为64B、stride192为64的倍数，初始库存成本不变，后续chunk仍按实际alignment公式收费。64B是布局粒度，不声称所有CPU cache line都是64B。
- 同类型字段组中的逗号表示依次排列；逐字段向其 alignment 上取整，最后按记录 alignment 上取整得到槽位。表内偏移只相对于该记录。
- 不使用 packed 结构、位域或指针低位编码压缩这些基线记录；record kind、状态和红黑树颜色均显式存放。

公共 prefix 为 16 B：kind:u16@0、layout_version:u16@2、flags:u32@4、record_id:u64@8。record_id 在元数据控制慢路径分配，不能通过未经证明的 64 位原子发号；不会回绕重用。它用于运行时诊断/身份，不是持久 allocation ID。

## 2. 槽位总表

下表字段末端包含中间对齐空洞，不包含尾部填充。每次 pool 分配按槽位扣账；空记录也不是免费的。

| 记录 | 字段末端 64 位 | 字段末端 32 位 | 槽位 64 位 B | 槽位 32 位 B |
| --- | --- | --- | --- | --- |
| ArenaHeader | 248 | 184 | 256 | 192 |
| ExtentDesc | 134 | 94 | 144 | 96 |
| LargeDesc | 88 | 76 | 96 | 80 |
| MappingDesc | 104 | 92 | 112 | 96 |
| BackendUnit | 112 | 92 | 112 | 96 |
| ThreadCtx | 128 | 104 | 128 | 128 |
| RangeTxn | 136 | 112 | 144 | 112 |
| TxnStep | 80 | 72 | 80 | 80 |
| RTPool | 136 | 112 | 192 | 128 |
| RTClass | 80 | 76 | 80 | 80 |
| RTBinding | 72 | 60 | 80 | 64 |
| PoolFamily | 64 | 48 | 64 | 48 |
| PoolChunk | 64 | 48 | 64 | 48 |
| HeapHeader | 64 | 56 | 64 | 64 |
| SlabDesc（分区布局） | 160 | 152 | 192 | 192 |
| RadixNode | 2080 | 1056 | 2080 | 1056 |
| PageTile，P=4KiB | 160 | 96 | 160 | 96 |
| PageTile，P=16KiB | 64 | 48 | 64 | 48 |
| Heap，44 个 class | 1536 | 832 | 1536 | 832 |

上述槽位不包含所属 PoolChunk 头部、页取整、空闲未用记录、可选诊断表或 OS 内核资源；这些在预算文件逐项计入。

## 3. ArenaHeader 与固定控制块

| 字段（列出的同类字段按顺序连续排列） | 类型 / 数量 | 64 位偏移 | 32 位偏移 |
| --- | --- | --- | --- |
| prefix | prefix | 0 | 0 |
| uuid | u64 × 2 | 16 | 16 |
| capacity | u64 | 32 | 32 |
| base | ptr | 40 | 40 |
| incarnation | u64 | 48 | 48 |
| page\_size,granularity,tile\_size,slab\_size | u32 × 4 | 56 | 56 |
| mode | u32 | 72 | 72 |
| health | a32 | 76 | 76 |
| broker\_root,free\_index,meta\_control,index\_root,shared\_heap,ctx\_list,central\_tail,lifecycle,vm\_ops,vm\_context,persist\_control,rt\_pool\_list,registry\_prev,registry\_next,pending\_head,retire\_tail | ptr × 16 | 80 | 80 |
| next\_owner\_token | ptr | 208 | 144 |
| next\_record\_id,next\_operation\_id | u64 × 2 | 216 | 152 |
| radix\_depth,ctx\_count,pending\_count,pool\_family\_count | u32 × 4 | 232 | 168 |

字段结束偏移：248 / 184；记录槽位：**256 / 192 B**（64 / 32 位）。未使用的尾部属于 padding，不可据此承诺可用扩展字段。

Header建立后capacity、base、incarnation、geometry与模式不改变。health是a32；其余可变计数在生命周期、metadata或broker指定的慢路径锁下访问，不能把裸u64当作32位原子。next_owner_token在登记控制下、next_record_id在元数据控制下递增，身份额度耗尽拒绝新身份。next_operation_id兼作broker revision，正常值不超过UINT64_MAX−1；begin和最终publish各消耗一代，并通过pending_count预留收口信用。UINT64_MAX为饱和哨兵：分页统一STALE_CURSOR，合法free/既有事务收口/cleanup仍继续，未编号清理opid为0。完整规则见[操作协议](operation-protocols.md)，不以诊断计数耗尽使已释放对象复活。

16 个 ptr 是控制结构引用，不表示它们存在于区外。vm_ops 可以指向库的只读函数表；vm_context、所有可增长列表及可选持久/RT 控制块均在 arena。未启用功能的指针为空。

pending_head在broker控制下挂接进行中事务。central_tail和retire_tail在central控制下分别锚定循环单链FIFO责任队列；空队列为NULL，非空队头为link_next(tail)。中央队列管理可接管slab；退休队列管理Slab/Large/MappingDesc、RTPool及终态隔离RangeTxn。link_next按prefix.kind选择普通记录的retire_next或终态RangeTxn的active_next。相同字段不能同时属于两个责任队列，不在free时创建节点。跳过或失败的可重试记录重新入队尾；真正OS工作前先独占claim并离开队列，不能持central锁做OS操作。v0.3将原central_list/retire_head同宽改名为central_tail/retire_tail，不增加指针。LimitBlock与StatsBlock位于固定控制块顺序中，通过编译期偏移定位，不再占Header指针；64位偏移为576/1024，32位为512/960。它们不受后续可变库存和页大小影响。

| 固定控制块 | 字段与槽位 | 访问责任 |
| --- | --- | --- |
| ControlLocks | lifecycle、broker、metadata_pool、metadata_growth、central 共 5 个 LockSlot，每个64B | 按架构锁序；shared heap 另带自己的 LockSlot |
| LockSlot | 64B、至少16B对齐，平台原位放置已验证可容纳的锁 | 不动态分配锁包装；原生锁不适配时重新预算/拒绝，不能溢出 |
| LimitBlock | prefix16B；12个条目，每条 limit/used/reserved/peak 各u64，共400B，按64B取整为448B | 慢路径预付/归还额度，不在每对象快路径更新 |
| StatsBlock | prefix16B；valid_mask:u64、snapshot_epoch:u64；20个u64值，共192B | 低成本采样或协调快照；不得无同步读取活 owner 字段 |
| FreeIndex | prefix16B；128位非空bitmap；128组 head/tail ptr；64B取整后2112/1088B | broker 独占维护 |
| MetaPoolControl | 固定64B根块 + 16个PoolFamily；64B取整后1088/832B | metadata pool/growth 协议 |
| BackendControl | 固定256B适配预算槽，含已核算的native状态/能力和reservation锚点 | 各平台必须给出实际占用；大状态移入已计费区内记录 |
| SizeClassGeometry | prefix16B + 44组 stride:u32、slot_count:u32，共368B，按64B取整为384B | 创建后只读 |

MetaPoolControl 的64B根块包含 prefix、chunk总链头、growth事务指针、growth_epoch:u64、family_count:u32、emergency_state:u32 和保留填充。16个family槽预留给已定义类型和专用数组，不表示所有family在创建时就分配记录。

12个limit槽分别表示 backend资源、runtime metadata、ctx数量、private span容量、empty span保留、fine views、ordinary views、pending事务、RT payload、roots、journal、checkpoint；[S-02](configuration.md)定义候选值和未启用值的解释。容量C是Header契约，不与后端资源额度混同。普通模式最后三个条目不造成日志或根表分配；持久R整段划入metadata地址used，未启用RESERVED子态用已有state/flags表达，不扩大记录槽位。

StatsBlock 的20个槽依次用于分配/释放事件、累计请求字节、已接收remote事件、live slot字节、live large字节、private span、empty span、metadata使用、metadata预留、broker free、最大兼容free、mapped字节、ordinary views、fine views、backend charge、resident采样、pending、quarantined和活ctx数量。valid_mask 标识本快照哪些项有效；没有跟踪能力的项不能填0伪装精确统计。

## 4. 区间与后端记录

### 4.1 ExtentDesc

| 字段（列出的同类字段按顺序连续排列） | 类型 / 数量 | 64 位偏移 | 32 位偏移 |
| --- | --- | --- | --- |
| prefix | prefix | 0 | 0 |
| start,length | u64 × 2 | 16 | 16 |
| owner,backend,pending | ptr × 3 | 32 | 32 |
| rb\_left,rb\_right,rb\_parent | ptr × 3 | 56 | 44 |
| free\_prev,free\_next | ptr × 2 | 80 | 56 |
| owner\_prev,owner\_next | ptr × 2 | 96 | 64 |
| backing\_offset | u64 | 112 | 72 |
| protection | u32 | 120 | 80 |
| owner\_kind,backing\_state | u16 × 2 | 124 | 84 |
| control\_pins | u32 | 128 | 88 |
| rb\_color,substate | u8 × 2 | 132 | 92 |

字段结束偏移：134 / 94；记录槽位：**144 / 96 B**（64 / 32 位）。未使用的尾部属于 padding，不可据此承诺可用扩展字段。

owner 指向 SlabDesc/LargeDesc/MappingDesc/RT控制块或对应 metadata owner；FREE 没有对象owner。backend 指向实际后端单元，pending 指向操作事务，二者不同。QUARANTINED范围的pending还可保留已经终结的RangeTxn恢复证据；这不表示仍在active Pending链中，不占进行中Pending名额。判定正在执行必须同时验证事务状态和active链归属，不能只检查指针非空。rb_* 只在地址树中使用；FREE时free_prev/free_next用于空闲索引，QUARANTINED时互斥复用为终态X.cleanup_list的关联extent双链，不能同时属于两者；owner_* 串联同一mapping/metadata owner的多个extent。

start/length 是 arena-relative范围；backing_offset 是该范围关联的后备偏移，不可推导为永久等于start。control_pins 由控制面锁管理，不在合法小对象free路径增加全局引用。

### 4.2 BackendUnit

| 字段（列出的同类字段按顺序连续排列） | 类型 / 数量 | 64 位偏移 | 32 位偏移 |
| --- | --- | --- | --- |
| prefix | prefix | 0 | 0 |
| start,length,file\_offset,committed\_charge | u64 × 4 | 16 | 16 |
| native\_handle,section\_handle,parent,prev,next | ptr × 5 | 48 | 48 |
| kind,flags,reference\_count,protection,pins,state | u32 × 6 | 88 | 68 |

字段结束偏移：112 / 92；记录槽位：**112 / 96 B**（64 / 32 位）。未使用的尾部属于 padding，不可据此承诺可用扩展字段。

BackendUnit 表示private allocation、view或共享section等OS单元，**不另行取得地址所有权**。多个views共享一个section时，用parent指向有引用计数的后备记录；共享父记录也要计费。native_handle/section_handle 没有使用时置无效，不能把同一native资源按每view重复关闭。

引用计数在后端控制慢路径中同步，不要求宽CAS。start/length与内部extent不一定一一对应；可拆/替换必须同时满足extent和backend边界。统计committed_charge不等于resident。

### 4.3 MappingDesc

| 字段（列出的同类字段按顺序连续排列） | 类型 / 数量 | 64 位偏移 | 32 位偏移 |
| --- | --- | --- | --- |
| prefix | prefix | 0 | 0 |
| mapping\_id,generation,original\_start,original\_length,live\_bytes | u64 × 5 | 16 | 16 |
| fragment\_head,backend\_list,retire\_next | ptr × 3 | 56 | 56 |
| profile,sharing,protection,state,fragment\_count,control\_pins | u32 × 6 | 80 | 68 |

字段结束偏移：104 / 92；记录槽位：**112 / 96 B**（64 / 32 位）。未使用的尾部属于 padding，不可据此承诺可用扩展字段。

fragment_head 管理存活extent；原始范围仅供查询，不授予已经归还洞的权限。fragment_count 受配额和u32边界限制。mapping_id/generation用于当前生命周期身份，root不得引用此会话记录。整体释放所需控制身份从创建时保留，不能临时向系统堆申请。

## 5. 对象、线程与heap

### 5.1 SlabDesc：三个64B分区

| 分区及相对偏移 | 字段 | 64/32位分区内偏移 |
| --- | --- | --- |
| cold，起点0 | prefix；extent ptr；span_offset:u64 | prefix@0；extent@16；offset@24（32位中间有4B空洞） |
| cold | span_bytes、slot_stride、slot_count:u32；class_id、flags:u16 | 32、36、40、44、46 |
| cold | requested_size_table ptr；retire_next ptr | 48/48、56/52；分区末端64/56 |
| owner，起点64 | freelist、backlog、class_prev、class_next、current_heap ptr | 0、W、2W、3W、4W |
| owner | live_count、free_count、bump_cursor、owner_state:u32 | 40/20、44/24、48/28、52/32 |
| owner | allocation_events:u64 | 56/40；分区末端64/48 |
| remote，起点128 | remote_head:aptr；owner_token:aptr | 0、W |
| remote | lifecycle_flags:a32；control_pins:u32；retire_epoch:u64 | 16/8、20/12、24/16；分区末端32/24 |

整体192B；owner独占本地字段，生产者只使用稳定身份和remote入口。live_count晚扣、发布后的最后访问规则保持不变。retire_next使空slab能在没有新事务记录时进入待回收队列；物理回收取得资源后再处理。

span_bytes/slot_count只服务64KiB slab，u32足够；更大对象不能滥用这些字段。requested_size_table默认为空，启用精确请求统计/诊断后按需另外分配，见预算。

### 5.2 LargeDesc

| 字段（列出的同类字段按顺序连续排列） | 类型 / 数量 | 64 位偏移 | 32 位偏移 |
| --- | --- | --- | --- |
| prefix | prefix | 0 | 0 |
| extent,backend | ptr × 2 | 16 | 16 |
| user\_offset,requested\_size,usable\_size,alignment,allocation\_id | u64 × 5 | 32 | 24 |
| retire\_next | ptr | 72 | 64 |
| control\_pins,state | u32 × 2 | 80 | 68 |

字段结束偏移：88 / 76；记录槽位：**96 / 80 B**（64 / 32 位）。未使用的尾部属于 padding，不可据此承诺可用扩展字段。

大对象记录精确requested/usable/alignment和合法返回位置。**快速索引只登记user_offset所在页**，free/usable_size/realloc必须匹配合法返回指针；内部指针不是合法调用。整个extent仍由broker保护，控制面地址所属查询走受保护范围树，不要求逐页建立LargeDesc指针。

这使一个超大连续对象的索引成本取决于返回地址所在tile，而不是对象字节数。小对象slab及RT仍覆盖全部可能出现合法对象起点的页。

### 5.3 ThreadCtx

| 字段（列出的同类字段按顺序连续排列） | 类型 / 数量 | 64 位偏移 | 32 位偏移 |
| --- | --- | --- | --- |
| prefix | prefix | 0 | 0 |
| arena,heap,owner\_token | ptr × 3 | 16 | 16 |
| thread\_cookie,arena\_incarnation | u64 × 2 | 40 | 32 |
| prev,next,tls\_next,rt\_bindings | ptr × 4 | 56 | 48 |
| rt\_binding\_count | u32 | 88 | 64 |
| state | a32 | 92 | 68 |
| allocation\_calls,free\_calls,requested\_bytes\_total,maintenance\_calls | u64 × 4 | 96 | 72 |

字段结束偏移：128 / 104；记录槽位：**128 / 128 B**（64 / 32 位）。未使用的尾部属于 padding，不可据此承诺可用扩展字段。

ctx属于一个线程和一个arena incarnation；owner_token与thread_cookie不是可复用的ctx地址。四个u64统计由当前owner更新或协调读取；requested_bytes_total是**累计事件字节**，不是精确存活请求字节。

rt_bindings是区内登记链，free热路径不遍历它搜索权限；RT prepare返回与当前ctx/pool绑定的稳定binding，调用时显式携带或使用预先选择的binding。

### 5.4 Heap与ClassState

HeapHeader为64B，依次为prefix、owner_token/arena ptr、maintenance_class_cursor/state/slab_count/class_count四个u32、private_capacity/retained_capacity两个u64；字段末端64/56B。后接一个64B LockSlot，再接44个ClassState。

ClassState为 active_slab ptr、owned_ring_head ptr、cached_count:u32、flags:u32，按16B取整为32/16B。因此Heap为：

- 64位：64 + 64 + 44×32 = **1536B**。
- 32位：64 + 64 + 44×16 = **832B**。
- 每登记一个普通线程，ctx+heap的记录成本为 **1664/960B**，尚未取得任何payload slab。

v0.3文档细化把此前未分配用途的HeapHeader.flags同宽改名为maintenance_class_cursor，并把partial_head明确为owned_ring_head；没有新增字段、位打包或字节成本。cursor取0..43，由heap owner或shared锁独占访问，不借用会饱和的统计计数。每类owned ring通过SlabDesc.class_prev/class_next包含该heap拥有的全部slab，包括full且等待remote归还的slab；active_slab只是同一环内的分配缓存引用，不是另一份所有权。维护访问后轮转类cursor与环头，摘除/移交必须先修正这些引用；具体进展条件见[争用与吞吐](contention-throughput.md)。

heap锁仅用于shared fallback及控制面；ctx拥有的本地快路径不因此加锁。detach移交所有slab之后才归还ctx和heap，SlabDesc不会随线程退出失效。共享heap从创建时预建，没有绑定一个用户ctx。

## 6. Pool与稀疏目录

### 6.1 PoolFamily / PoolChunk

| 字段（列出的同类字段按顺序连续排列） | 类型 / 数量 | 64 位偏移 | 32 位偏移 |
| --- | --- | --- | --- |
| free\_chunks,full\_chunks,retired\_chunks,emergency\_head | ptr × 4 | 0 | 0 |
| stride,record\_alignment,chunk\_count,flags | u32 × 4 | 32 | 16 |
| used\_records,reserved\_records | u64 × 2 | 48 | 32 |

字段结束偏移：64 / 48；记录槽位：**64 / 48 B**（64 / 32 位）。未使用的尾部属于 padding，不可据此承诺可用扩展字段。

free/full/retired为chunk链；emergency_head单独管理禁止普通申请消耗的应急记录。stride和alignment在初始化时固定。reserved_records表示应急库存，used_records和普通空闲数必须与chunk计数一致，不能将应急记录同时放入普通free链。

| 字段（列出的同类字段按顺序连续排列） | 类型 / 数量 | 64 位偏移 | 32 位偏移 |
| --- | --- | --- | --- |
| prefix | prefix | 0 | 0 |
| extent,family,prev,next | ptr × 4 | 16 | 16 |
| slot\_count,free\_count,first\_free,flags | u32 × 4 | 48 | 32 |

字段结束偏移：64 / 48；记录槽位：**64 / 48 B**（64 / 32 位）。未使用的尾部属于 padding，不可据此承诺可用扩展字段。

PoolChunk后按family alignment对齐放置记录。空闲记录内可存next索引/指针；不额外为每条记录创建外部节点。first_free使用u32局部索引，chunk不允许包含超过u32可表示的记录数。bootstrap中的多个family可以共享同一次OS提交，但各自有chunk头；后续growth可在同一metadata extent内切出多个family chunk以补足应急记录。

可变长度的RT状态数组、精确requested表、诊断位图和连续事务工作区不通过普通malloc分配。首版采用独立、按后端单元取整的metadata buffer，使用同形LargeDesc槽位但不同record kind管理其长度与归还；metadata只走内部buffer接口，不加入用户对象索引。可连续容纳时也可从尚未切分的bootstrap尾部取得；这种尾部切块只在quiescent整块重置时合并回收。其记录、Extent/BackendUnit、取整和暂留空间全部计费，不能只报数组净字节。更复杂的小buffer复用是后续优化，不引入未计费的隐藏子分配器。

### 6.2 RadixNode与PageTile

二者头部均为32B：prefix@0、prefix/tile_index:u64@16、level或entry_count:u32@24、occupied_count:u32@28。RadixNode后接256个aptr，因此2080/1056B。PageTile后接T/P个aptr，槽位为 align16(32+(T/P)×W)。

目录根只包含已发布子指针，空指针代表不存在路径；root创建时存在，其他节点按需建立。动态路径发布在broker慢路径串行化，读取方acquire，目录节点仅在quiescent时回收。metadata及普通文件map不走用户对象free定位，**不为metadata自举建立对象PageTile**，避免自描述递归。

从预留容量算深度：D=max(1,ceil(log256(ceil(C/T))))；16MiB时D=1，4TiB时D=4。节点大小、D和PageTile不是随C预分配的密集数组。

## 7. 事务记录与归还责任

### 7.1 RangeTxn

| 字段（列出的同类字段按顺序连续排列） | 类型 / 数量 | 64 位偏移 | 32 位偏移 |
| --- | --- | --- | --- |
| prefix | prefix | 0 | 0 |
| operation\_id,target\_generation,start,length | u64 × 4 | 16 | 16 |
| arena,target,steps,extent\_credits,index\_credits,cleanup\_list,active\_next | ptr × 7 | 48 | 48 |
| state,flags,step\_count,completed\_count | u32 × 4 | 104 | 76 |
| native\_error | u64 | 120 | 96 |
| status,reserved | u32 × 2 | 128 | 104 |

字段结束偏移：136 / 112；记录槽位：**144 / 112 B**（64 / 32 位）。未使用的尾部属于 padding，不可据此承诺可用扩展字段。

target和steps中的旧对象/后端记录在操作完成前保持pin；需要的新extent/index credit已经预取。active_next挂入进行中操作列表；终态隔离时先退出该链并撤销Pending，之后才互斥复用为retire_tail队列next。cleanup_list在终态仅存关联Q extent，以其free_prev/free_next维护；孤立backend责任由steps.old_unit/new_unit保留，不混入extent链。completed_count在终态清理阶段重置为扫描游标，逐步仍按effect判断，不把计数当成功前缀。最后关联extent及后端责任全部解除前X/J不能回收；同一后端清理权不能同时属于X和原owner退休记录。state记录preflight/pending/backend/publish/rollback/quarantine阶段，具体转换和受保护读写见[状态机](state-machines.md)。

### 7.2 TxnStep

| 字段（列出的同类字段按顺序连续排列） | 类型 / 数量 | 64 位偏移 | 32 位偏移 |
| --- | --- | --- | --- |
| start,length,file\_offset | u64 × 3 | 0 | 0 |
| old\_unit,new\_unit,extent | ptr × 3 | 24 | 24 |
| old\_prot,new\_prot,effect,flags | u32 × 4 | 48 | 36 |
| expected\_generation,native\_error | u64 × 2 | 64 | 56 |

字段结束偏移：80 / 72；记录槽位：**80 / 80 B**（64 / 32 位）。未使用的尾部属于 padding，不可据此承诺可用扩展字段。

Step没有prefix，属于RangeTxn拥有的数组/链式固定块。每步覆盖一个可独立描述的backend变更单元，effect明确区分未执行、已改变、已恢复与不确定；旧protection和expected_generation不是用户数据的备份。

需要大批steps时从区内metadata分配完整记录块或链，操作中不得临时扩容。故障报告遍历这些稳定记录；正式操作必须在用户范围OS修改前确认caller report足以一次装下全部本次片段，不足返回BUFFER_TOO_SMALL。查询分页另按cursor/revision协议，不替代本次失败报告。

VOLATILE合法free可以先完成逻辑归还，使用已有Slab/LargeDesc的retire_next登记后端清理。没有RangeTxn库存时延后OS清理，而不是使已经释放的对象复活。部分unmap/protect等新操作则可以在修改前报告METADATA_LIMIT。

## 8. RT控制记录

| 字段（列出的同类字段按顺序连续排列） | 类型 / 数量 | 64 位偏移 | 32 位偏移 |
| --- | --- | --- | --- |
| prefix | prefix | 0 | 0 |
| arena,owner\_ctx,classes,states,bindings,retire\_next | ptr × 6 | 16 | 16 |
| slot\_count,cursor,payload\_offset,payload\_bytes | u64 × 4 | 64 | 40 |
| registered\_count,class\_count,scan\_budget,state | u32 × 4 | 96 | 72 |
| owner\_token | ptr | 112 | 88 |
| allocation\_events,collected\_returns | u64 × 2 | 120 | 96 |

字段结束偏移：136 / 112；记录槽位：**192 / 128 B**（64 / 32 位）。未使用的尾部属于 padding，不可据此承诺可用扩展字段。

RTPool.retire_next在LIVE时作为Header.rt_pool_list的单链next；逻辑销毁在central保护下先摘活动链，再复用该字段入retire_tail队列，绝不同时处于两链。classes/states/bindings仍保留清理所需所有权，物理回收完成前不复用RP/RC；不增加槽位或动态退休节点。

RTPool的最后两个u64是owner更新的分配事件和**已接收归还事件**；生产者free不修改聚合统计。slot_count/cursor为owner-only u64，即使Android32也无需64位原子。状态数组仍是每slot一个a32。

| 字段（列出的同类字段按顺序连续排列） | 类型 / 数量 | 64 位偏移 | 32 位偏移 |
| --- | --- | --- | --- |
| prefix | prefix | 0 | 0 |
| pool | ptr | 16 | 16 |
| start,length,stride,slot\_count,state\_first | u64 × 5 | 24 | 24 |
| freelist | ptr | 64 | 64 |
| class\_id,flags | u32 × 2 | 72 | 68 |

字段结束偏移：80 / 76；记录槽位：**80 / 80 B**（64 / 32 位）。未使用的尾部属于 padding，不可据此承诺可用扩展字段。

每个class/页块种类取得自己的P对齐范围，不在同一页混放不同RTClass。这样PageTile直接定位RTClass，再计算slot下标；freelist仅owner操作。state_first是该类在pool总状态数组中的起始下标。

| 字段（列出的同类字段按顺序连续排列） | 类型 / 数量 | 64 位偏移 | 32 位偏移 |
| --- | --- | --- | --- |
| prefix | prefix | 0 | 0 |
| pool,ctx,owner\_token | ptr × 3 | 16 | 16 |
| arena\_incarnation,pool\_id | u64 × 2 | 40 | 32 |
| next\_in\_ctx | ptr | 56 | 48 |
| state,reserved | u32 × 2 | 64 | 52 |

字段结束偏移：72 / 60；记录槽位：**80 / 64 B**（64 / 32 位）。未使用的尾部属于 padding，不可据此承诺可用扩展字段。

一个binding对应一个pool与一个已登记ctx；prepare阶段生成并冻结，run阶段校验固定字段，不搜索全局arena或遍历可增长登记表。pool/context/代次仍需共同匹配。quiescence之后才能销毁binding，不做运行时撤销后继续free的保证。

RTBinding.owner_token是所绑定ctx的身份，可能是远程归还线程；它不要求等于RTPool.owner_token。只有alloc/maintenance要求调用者是pool owner，合法free只要求稳定binding匹配当前pool/ctx及生命周期。

## 9. 编译验证与设计边界

S-01交付的是上述**条件明确的布局算式**。I-03/I-04需要逐个平台核对实际sizeof/alignof/offsetof、原子lock-free、LockSlot和BackendControl原位适配、每类chunk头与页取整；任何不符均先修预算或拒绝配置，不向区外借空间。

64B填充、每slot请求表、pool空置和事务工作区均有可见成本。持久superblock/log/checkpoint的精确字节格式属于S-05；本文件只计算其运行时元数据引用和工作区，不能据此声称16MiB持久实例已有完整空间证明。

