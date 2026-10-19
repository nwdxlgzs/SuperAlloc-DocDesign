# S-04：C ABI 详细规格草案

状态：**文档设计已给出确定选择，尚未生成头文件、编译布局或发布 ABI**。本文为后续 C17 头文件提供字段、数值、签名和失败合同；[API 契约](api-contracts.md)定义产品行为，[默认配置](configuration.md)定义配置值，[运行时布局](metadata-layout.md)不属于公开 ABI。本文的候选编号在正式冻结前可通过显式修订改变，不能把本文误当成已发布兼容版本。

## 1. 类型、调用约定与版本

- 库使用 C linkage；所有下列函数返回 `sa_status`，其表示类型为 `int32_t`，不跨边界抛异常。Windows ARMv7 不在目标中；Windows x64/ARM64 使用平台 C ABI，Android ARMv7 使用 NDK 默认 AAPCS，其他目标使用其平台 C ABI。
- `u8/u32/u64/i64` 在表中分别表示 `uint8_t/uint32_t/uint64_t/int64_t`。`size_t` 仅表示当前进程可寻址对象长度；容量与文件/arena offset 使用 u64。输出对象指针使用 `void **`，不通过整数返回指针。
- `sa_arena`、`sa_ctx`、`sa_rt_pool`、`sa_rt_binding` 是不透明、不完整类型；公开句柄为对应原生指针。调用者不分配其内部对象，不读取其布局。销毁后的裸 handle 不可再用，库不承诺安全检测任意陈旧地址。
- u64 字段明确要求 8 B 对齐，公共结构 alignment 至少 8 B；不使用 packed、位域、native long、C/C++ enum 或 bool 表示二进制字段。输出数组的每项起点亦需满足其 alignment。
- **ABI 版本** `0x00010000`：高 16 位为 major，低 16 位为 minor。首版 major=1、minor=0。当前只是该版本的冻结候选。
- 同一目标 ABI 内保证兼容；32 位、LP64 和 LLP64 不是可以互换调用的同一机器 ABI。含原生指针的结构分别给出布局；固定宽结构相同字段偏移不等于可直接跨进程或当磁盘格式使用。
- 第一版目标按各支持平台的小端 ABI 构建。公共值结构使用主机整数表示；持久文件按[磁盘格式](persistent-format.md)编码，禁止直接 dump 公共结构代替序列化。

### 1.1 可扩展结构的共同头

所有配置、请求和单个输出结构以 `struct_size:u32@0, abi_version:u32@4` 开头。表中列出的 size 是首版完整大小；无特殊说明时也是首版最小大小。

输入初始化为全零，再填头与所需字段。未知非零 flag、未知 option key、重复 key、非零 reserved 以及未实现的功能返回 `INVALID_ARGUMENT` 或 `UNSUPPORTED`，不忽略调用方意图。支持 major=1 且不高于库 minor 的版本；其他版本返回 `INCOMPATIBLE_ABI`。

`struct_size` 小于本版本最小大小返回 `BUFFER_TOO_SMALL`，不读取范围外字节；大于库已知大小时，输入额外字节必须为零。输出调用者先填写头；库只写已知字段且不越过声明范围，未知尾部保持原样。头中的 `struct_size` 保留调用者值。首版的 optional 尾部仅限各结构明确列出的 reserved 区；删除尾部必须仍满足表列最小大小，不能任意缩短到头部。

值类型 `sa_ref`、`sa_phandle`、`sa_range_handle`、`sa_native_file`、`sa_option` 及下表未列header的记录（包括sa_object_info）、buffer 内记录和数组元素没有版本头，大小由所属 ABI major 固定；未来新增更大表示需要新类型/接口或新的 major，不能静默扩大现有数组 stride。

## 2. 状态码与数值空间

| 数值 | `SA_` 后缀 | 含义 |
| --- | --- | --- |
| 0 | OK | 成功；具体持久阶段见模式及输出 |
| 1 | INVALID_ARGUMENT | 无效参数、未定义输入位、重复配置键 |
| 2 | OVERFLOW | 长度、乘法、取整或窄化不可表示 |
| 3 | UNSUPPORTED | 该实例/后端/模式不提供请求能力 |
| 4 | INCOMPATIBLE_FORMAT | 不支持的持久格式 |
| 5 | INCOMPATIBLE_GEOMETRY | 页或存储几何不兼容 |
| 6 | WRONG_ARENA | 有效输入属于不同实例/store |
| 7 | INVALID_HANDLE | 有效可检查记录的身份/代次不匹配 |
| 8 | COLLISION | 请求范围已由其他所有者占用 |
| 9 | BUSY | 生命周期、正在执行的范围操作或直接 root 引用阻止操作 |
| 10 | CLOSED | 仍有效控制对象处于关闭状态 |
| 11 | WRITE_FAULTED | durable 实例禁止继续写入 |
| 12 | NO_ADDRESS_SPACE | 不能保留所要求的连续虚拟范围 |
| 13 | OUT_OF_MEMORY | 普通分配可用资源不足 |
| 14 | NO_CONTIGUOUS_RANGE | 有空闲但没有兼容连续范围 |
| 15 | METADATA_LIMIT | 区内管理资源或增长额度不足 |
| 16 | COMMIT_LIMIT | 后端 commit 配额不足 |
| 17 | LIMIT | 其他配额或累计身份空间耗尽 |
| 18 | WOULD_BLOCK | RT 本次有限工作未取得所需 slot |
| 19 | OS_ERROR | 已知结果的非持久 OS 错误 |
| 20 | IO_ERROR | 已知结果的 I/O 错误 |
| 21 | PARTIAL_FAILURE | 范围修改部分成功/隔离，完整片段报告有效 |
| 22 | COMMIT_UNKNOWN | 持久提交结果不确定，进入 WRITE_FAULTED |
| 23 | CORRUPT | 检测到不符合格式或状态不变量 |
| 24 | QUARANTINED | 目标范围当前禁止复用 |
| 25 | BUFFER_TOO_SMALL | 输出/报告空间或结构头长度不足 |
| 26 | STALE_CURSOR | 枚举所依据的 revision 已变化 |
| 27 | NOT_FOUND | 指定目录 ID/空 root 无当前对象 |
| 28 | INCOMPATIBLE_ABI | ABI 版本不兼容 |
| 29 | REENTRANT | 检测到禁止的同实例回调重入 |

30～255 保留给同 major 的后续状态；负值和 256 以上本版不返回。调用方必须把未知非零状态视为失败，并查询通用错误字段，不能假设只有当前枚举值。

### 2.1 `sa_error`：128 B

| 字段 | 类型 | 偏移 |
| --- | --- | --- |
| header | 8 B | 0 |
| status, operation | u32 × 2 | 8、12 |
| operation_id | u64 | 16 |
| native_domain, phase | u32 × 2 | 24、28 |
| native_code | i64 | 32 |
| offset, length | u64 × 2 | 40、48 |
| health, flags | u32 × 2 | 56、60 |
| resource_key, reserved0 | u32 × 2 | 64、68 |
| limit, used, reserved_amount, requested_amount | u64 × 4 | 72、80、88、96 |
| reserved1 | u64[3] | 104 |

`native_domain`：0 无、1 POSIX errno、2 Win32、3 平台其他；`phase`：0 验参、1 预检、2 资源准备、3 Pending、4 后端执行、5 发布、6 回滚、7 持久提交、8 恢复。health：0 READY、1 WRITE_FAULTED、2 CLOSING、3 CLOSED；范围隔离使用 flags bit 0，可能存在可见修改 bit 1，提交结果未知 bit 2。

每个签名的最后一个`sa_error *e`可为空；不为空时必须预先初始化合法头。错误结构校验在副作用前完成。RT调用仅写固定字段，不做字符串格式化、系统error查询或callback。普通fast path的operation_id可以为零；正常VM/durable慢路径使用不回绕的operation ID，不为每次malloc添加全局发号原子。runtime诊断revision饱和后的清理opid也为0（未编号）；清理责任不依赖opid唯一性。

配额错误使用resource_key及其limit/used/reserved_amount/requested_amount，描述本次校验观察到的阻塞，不承诺它仍是下一次调用时的全局状态。非配额错误这些字段为零；所有reserved输出为零。

`operation` 使用后文函数目录的编号。状态 OK 时 e 若存在被写成成功诊断；输出头本身无效时只能可靠返回 status，不能越界回填 e。

status描述本调用已确认的结果，health描述实例采样状态，二者不是同一个状态机。已确认durable commit并完成runtime发布的调用可以返回OK且health=WRITE_FAULTED（由并发主文件I/O锁存）；不得把确定成功改报COMMIT_UNKNOWN诱导重试。checkpoint镜像已知部分发布时，内部admission可在health=READY下保持关闭，新修改返回BUSY，按[API故障合同](api-contracts.md)显式继续checkpoint或退出。范围隔离也不自行改变上述health枚举。本轮不新增状态码、字段或函数。

## 3. 身份和值结构

### 3.1 跨会话持久引用

| 类型/字段 | 类型 | 偏移 | 总大小 |
| --- | --- | --- | --- |
| `sa_ref.store_uuid` | u8[16] | 0 | 40 B |
| allocation_id | u64 | 16 | |
| generation | u64 | 24 | |
| byte_delta | u64 | 32 | |
| `sa_phandle.store_uuid` | u8[16] | 0 | 64 B |
| allocation_id, generation | u64 × 2 | 16、24 | |
| offset, requested_length, usable_length | u64 × 3 | 32、40、48 | |
| flags, reserved | u32 × 2 | 56、60 | |

allocation_id 是主文件目录槽 ID，generation 在该槽复用时递增，达到最大值后拒绝复用。offset/length 是已验证对象的结果信息，不授权调用方修改身份。phandle 的 flags、reserved 本版输出为零。

所有字节为零的 ref 是空引用；root_set 用它清空 root，root_get 对空 root 返回 `NOT_FOUND` 并输出零 ref。非空 ref 必须 UUID/ID/generation 全部有效；不能仅凭 offset 解析。`byte_delta < requested_length`，解析的访问长度还需满足 checked `byte_delta + length <= requested_length`；不提供 one-past 引用。根仅允许 byte_delta=0。

持久引用的逻辑含义可供应用自行序列化；结构内的整数不是通用磁盘编码承诺。phandle 转 ref 由调用者复制 UUID/ID/generation 并填入合法 delta，无需隐式全局查询。

### 3.2 会话内范围身份

`sa_range_handle` 为 40 B：`arena_uuid:u8[16]@0, incarnation:u64@16, resource_id:u64@24, generation:u64@32`。`sa_mapping` 和 `sa_page_lease` 是语义别名，运行时校验实际记录类型。全零表示无效。

句柄不能延长生命周期。部分 unmap 后 survivors 保留原身份；revision 改变，hole 新所有者有新 resource_id。完整解除后该 handle 失效。所有操作同时接收显式 arena，避免全局地址路由。ID/generation/incarnation 不回绕，耗尽返回 LIMIT。

### 3.3 `sa_native_file`：16 B

`kind:u32@0, flags:u32@4, value:u64@8`；kind=0 NONE、1 POSIX_FD、2 WINDOWS_HANDLE，flags 首版为零。POSIX value 必须是非负且可转换到 int 的 fd；Windows value 是 uintptr_t 零扩展后的 HANDLE，不接受 INVALID_HANDLE_VALUE 或伪句柄。跨平台错误 kind 返回 UNSUPPORTED。

库在成功映射/open/create 前复制所需 native 引用。调用方在调用期间保持输入有效，成功后可关闭原引用；失败不关闭调用方引用。value 从不写入持久文件。

open可在控制岛建立前通过**借用的输入file**只读少量固定header以确定容量/几何，既不写文件也不取得库自有fd。reserve/bootstrap完成后再dup、取得独占lease并重新读取/完整验证；前次预读不是可信恢复结果，前后文件变化不能绕过第二次验证。

### 3.4 初始化清理凭据 `sa_cleanup_token`：128 B

| 字段 | 类型 | 偏移 |
| --- | --- | --- |
| header | 8 B | 0 |
| kind, flags | u32 × 2 | 8、12 |
| base, capacity, control_offset, control_length | u64 × 4 | 16、24、32、40 |
| cleanup_nonce | u64 | 48 |
| native_kind, phase | u32 × 2 | 56、60 |
| native_reservation, incarnation | u64 × 2 | 64、72 |
| reserved | u64[6] | 80 |

kind：0 EMPTY、1 PRE_CONTROL_RESERVATION、2 CONTROL_ISLAND；flags bit0 UNKNOWN_NATIVE_ID表示含不能安全重试的原生身份诊断，其他零。native_kind：0无、1POSIX范围、2Windows reservation；native_reservation为后端固定宽编码，调用者不能解释或关闭它。base/native值须能窄化到当前ABI指针宽度。nonce/incarnation只帮助诊断，不能使复制token后的重复释放合法。

create/open/init输入的cleanup输出必须已初始化且kind=EMPTY。一般成功或已完整清理的失败保持EMPTY；初始化失败且仍有**明确可重试**的自有资源时返回PARTIAL_FAILURE，out_arena仍为NULL，token非空，由调用方保存直到sa_cleanup_retry成功清空。

destroy/discard/close/abort-close也要求EMPTY cleanup输出。验参、BUSY等尚未进入清理的失败保持原arena有效；一旦开始不可逆资源拆卸，旧arena不再作为调用入口。未清完的可重试资源转入token，后续只能cleanup_retry，不能同时用旧arena再close。最后控制岛可能已经不可访问，库不得为了输出错误再解引用它。

close若在持久checkpoint/clean发布阶段失败、尚未开始runtime/native资源拆卸，原arena仍是诊断和abort-close入口；提交不明时其状态为WRITE_FAULTED，cleanup保持EMPTY。这与资源拆卸已经开始后的token接管是两个不同阶段，不能仅看到非OK就重试同一个close。

控制岛可写以前不打开文件、不取得无界资源；PRE_CONTROL只描述一个reservation及单个bootstrap启用范围。控制岛已建立后的余留资源由其区内清单描述，CONTROL_ISLAND token仅定位该cleanup-only控制岛，不能用于分配。token本体是调用者提供的固定输出，不能扩展或借外部heap；整个token只允许转移所有权，不允许复制后并发/重复cleanup。

清理重试只针对操作语义确认仍拥有的资源。POSIX close(EINTR)之类可能已关闭或句柄已失去所有权的情况，在e和UNKNOWN_NATIVE_ID诊断状态中报告，**不把可被系统复用的fd盲目存入可执行重试列表**。retry可以推进其他确定清理，但身份不确定尚未收口时返回OS_ERROR，不能伪称全部责任已经结束；相关诊断记录保留在既有控制岛。借用buffer不解除调用者映射，初始化失败后的已触及控制字节不承诺恢复原内容，cleanup通常EMPTY。

## 4. 配置、选项和缓冲区

### 4.1 `sa_arena_config`：48 B

| 字段 | 类型 | 偏移 | 规则 |
| --- | --- | --- | --- |
| header | 8 B | 0 | 最小/完整均 48 |
| mode | u32 | 8 | 0 VOLATILE、1 CLEAN_REOPEN、2 RECOVERABLE_METADATA |
| defaults_version | u32 | 12 | 首版必须 1 |
| capacity | u64 | 16 | create 精确容量；open 为期望容量，0 表示采用文件记录值 |
| flags | u64 | 24 | bit0 TRACK_REQUESTED、bit1 DIAGNOSTIC_CHECKS；其他拒绝 |
| reserved | u64[2] | 32 | 全零 |

create 的 source 由 mode 与单独 file 参数确定：VOLATILE 要求 file 为 NONE；持久模式要求文件有效且是调用方已创建的空专用文件，库只在明确 create 路径建立格式。open 不格式化/截断。借用文件映射 buffer 通过 init_in_place，mode 必须 VOLATILE。

没有path参数：文件的新路径创建、目录项同步和namespace durability由调用方管理；库只承诺经验证的文件内容屏障，不提供namespace持久化flag/入口，不能从FILE_BARRIER推导目录项已经安全。open options中1002/1010/1011/1012/1106若显式出现，必须等于已验证文件的R/Q/J/2K/N，否则INCOMPATIBLE_GEOMETRY；缺少这些key采用文件值，不重新套默认公式改写布局。其他允许的运行额度按configuration校验，不能降低到重建所需以下。

### 4.2 `sa_option`：16 B

`key:u32@0, flags:u32@4, value:u64@8`。flags 本版全零。函数接收 options 指针和 u32 count，count=0 时指针可为空；数组不重复，不借用其生命周期。缺少 key 使用 defaults_version 对应默认值，显式 value=0 是真实零值/禁用值，不能再解释为 AUTO。

| key | 名称 | 单位 |
| --- | --- | --- |
| 1001 | BACKEND_LIMIT | B |
| 1002 | RUNTIME_METADATA_LIMIT | B |
| 1003 | CTX_LIMIT | 个 |
| 1004 | PRIVATE_SPAN_LIMIT | B，实例汇总 |
| 1005 | EMPTY_SPAN_LIMIT | B，实例汇总 |
| 1006 | FINE_VIEW_LIMIT | 个 |
| 1007 | ORDINARY_VIEW_LIMIT | 个 |
| 1008 | PENDING_LIMIT | 个 |
| 1009 | RT_PAYLOAD_LIMIT | B |
| 1010 | ROOT_LIMIT | 个 |
| 1011 | JOURNAL_LIMIT | B |
| 1012 | CHECKPOINT_LIMIT | B |
| 1101 | REMOTE_DRAIN_BUDGET | 个/次，必须 >0 |
| 1102 | RT_SCAN_DEFAULT | slot/次，必须 >0 |
| 1103 | FINE_VIEWS_PER_MAP | 个 |
| 1104 | PRIVATE_SPAN_PER_CTX | B |
| 1105 | EMPTY_SPAN_PER_CTX | B |
| 1106 | PERSISTENT_ALLOCATION_LIMIT | 个 |

首版全部额度在create/open时求值并保持不变；不导出运行时set_limits函数。维护预算逐次传入，RT的K在prepare时固定。取整方式和16MiB公式以configuration为准。1012是两个checkpoint容量**合计2K**，不是单份大小。private_span_per_ctx限制独占slab容量，达到上限可移交带live slab或走shared heap，不是线程活对象总量上限。

计数键1003、1006、1007、1008、1010、1103、1106上限均为UINT32_MAX；1101/1102为1..UINT32_MAX。0是否禁用仍依配置合同。字节键使用u64并受C、几何和模式约束；不能仅因字段是u64就接受超过后端范围的值。RT class_count、participant_count和每类slot_count均为1..UINT32_MAX；所有类总slot_count由checked u64求和并通过容量/metadata预算限制。root直接引用数因此可安全表示为u32。

### 4.3 `sa_buffer`：32 B，含原生指针

| 字段 | 类型 | 64 位偏移 | 32 位偏移 |
| --- | --- | --- | --- |
| header | 8 B | 0 | 0 |
| data | void* | 8 | 8 |
| capacity, written, required, stride | u32 × 4 | 16、20、24、28 | 12、16、20、24 |
| padding | 明确零填充 | 无 | 28～31 |

调用者设置 data/capacity/stride，written/required 初始化为零。stride 必须不小于对应记录大小且为其 alignment 倍数；多出的项内尾部不写。capacity=0 时 data 可空，stride 仍填写记录大小。库检查 count×stride 不溢出。

required/written和所有数组条数均最多UINT32_MAX。若计划`3k+2`或其他完整输出计数超过该上限，返回OVERFLOW且不进入Pending/用户范围OS变更，不截断required；调用方可把工作拆成独立范围操作。指针覆盖字节数还必须能由当前ABI的size_t表示，32位先检查再转换。

非分页查询空间不足返回 BUFFER_TOO_SMALL，written=0、required=所需完整条目数。分页查询每次至少需要一条容量，written 是本页条数，required 是继续操作的最小容量（未结束为 1，结束为 0），不为求精确剩余量遍历全部数据。

### 4.4 `sa_cursor`：64 B

`header@0, kind:u32@8, flags:u32@12, arena_uuid:u8[16]@16, incarnation:u64@32, resource_id:u64@40, revision:u64@48, position:u64@56`。kind：1 mapping、2 allocations、3 roots；flags bit0 DONE，其余零。

首次使用只设置头，其余置零；库写入kind/身份/revision。后续不得修改。枚举不固定后台snapshot，也不保存游标对应的增长记录；revision复用S01的broker mutation epoch/next_operation_id。每次Pending的begin及最终publish/rollback各推进一次，直接可见map/持久目录/root变更也推进；**无关范围变更也可以保守地使游标失效**。下一页revision不同返回STALE_CURSOR、written=0，调用方重置重启；arena存在任一Pending时分页保守返回BUSY。复制本页期间持broker保护，不在解锁后追踪可退休记录。

UINT64_MAX是revision饱和哨兵，所有上述分页（包括新游标第一页）统一返回STALE_CURSOR，此规则优先于Pending的BUSY；普通非分页诊断仍可用。正常事务按[操作协议](operation-protocols.md)用pending_count预留begin/end推进信用，无信用的新范围/身份创建在OS前LIMIT。合法VOLATILE free、完整page_free、RT逻辑销毁、既有事务收口与cleanup仍继续，必要时将revision饱和，不因诊断计数耗尽拒绝；未编号清理operation_id=0。身份generation、持久LSN/epoch/super代次绝不适用此豁免。普通slot热路径仍不更新该全局计数。

## 5. 映射请求与结果

### 5.1 固定宽请求

| 类型 | 字段顺序及偏移 | 大小 |
| --- | --- | --- |
| `sa_page_request` | header@0；length:u64@8；alignment:u64@16；placement:u32@24；flags:u32@28；offset:u64@32；reserved:u64@40 | 48 B |
| `sa_map_request` | header@0；length:u64@8；offset:u64@16；file_offset:u64@24；file:sa_native_file@32；placement/backing/sharing/protection:u32@48/52/56/60；split_policy/flags:u32@64/68；alignment:u64@72；reserved:u64[2]@80 | 96 B |
| `sa_range_request` | header@0；offset:u64@8；length:u64@16；flags:u32@24；protection:u32@28 | 32 B |
| `sa_resize_request` | header@0；new_length:u64@8；target_offset:u64@16；placement:u32@24；flags:u32@28；reserved:u64[2]@32 | 48 B |

placement：0 ANY、1 FIXED_NO_REPLACE；ANY 要求 offset/target_offset=0。backing：0 ANONYMOUS、1 FILE。sharing：0 PRIVATE、1 SHARED；匿名首版只接受 PRIVATE。split_policy：0 ORDINARY、1 PAGE_SPLITTABLE。protection 位：READ=1、WRITE=2、EXEC=4，零是 NOACCESS；组合由后端能力检查。

length、file_offset、fixed offset 必须 P 对齐，length>0；alignment=0 表示 P，否则是 >=P 的二次幂。page flags bit0 ZERO，其他拒绝；map flags 首版全零；range flags 首版全零，protection 只在 mprotect 有效，其他调用必须为零。resize flags bit0 MAY_MOVE：未设置时必须placement=FIXED_NO_REPLACE且target_offset等于当前完整映射起点；设置时可ANY+target_offset=0让库选址，或FIXED_NO_REPLACE+指定目标。目标等于原起点是原址路径；其他固定目标不得与原范围部分重叠。

range offset 始终是 **arena-relative**，不随 handle 当前首片段改变。replace 同时接收旧范围和新 map request：长度/固定 offset 必须相同，placement 必须 FIXED_NO_REPLACE；提供旧 handle 是允许替换自己范围的唯一授权，不是允许覆盖别的 owner。

### 5.2 `sa_map_result`：96 B

`header@0, flags:u32@8, phase:u32@12, operation_id:u64@16, mapping:sa_range_handle@24, base_offset:u64@64, length:u64@72, old_length:u64@80, reserved:u64@88`。

flags：bit0 MAPPING_VALID、bit1 HAS_PARTIAL_REPORT、bit2 HAS_QUARANTINE、bit3 OLD_BASE_INVALID。base_offset 是成功结果的起点；部分结果以片段报告为准，不能从 length 推导整个区间仍可访问。最后片段解除后 mapping 清零。mremap 保留映射逻辑身份，输出的新起点生效，旧地址可失效。

### 5.3 `sa_fragment`：64 B

`offset:u64@0, length:u64@8, old_resource_id:u64@16, new_resource_id:u64@24, old_backing:u32@32, new_backing:u32@36, old_protection:u32@40, new_protection:u32@44, outcome:u32@48, phase:u32@52, native_code:i64@56`。

outcome：0 UNCHANGED、1 APPLIED、2 ROLLED_BACK、3 QUARANTINED。backing 使用上述 0/1，2 表示 reserved/no-access 无 active backing。报告按 offset 升序、互不重叠，只描述本次操作涉及的范围，不重复输出不相关 survivors。

所有可能进行多片段 OS 修改的入口必须提供 report buffer。预检依据 [操作协议](operation-protocols.md)的 k 步计划计算 `3k+2` 保守上界；容量不足时在 Pending 和用户范围 OS 修改前返回 BUFFER_TOO_SMALL，并回填 required。空间充足才开始正式操作。因此 PARTIAL_FAILURE 时可以一次完整报告全部相关片段，不再事后申请元数据或隐式丢弃错误范围。

capacity=0 可用来获得预估，不预留资源；正式重试要重新验证计划，必要时再次无副作用返回 BUFFER_TOO_SMALL。没有长期保留的 operation-report 对象，也不提供根据旧 operation_id 读取历史的接口。

map_query 使用 cursor+buffer 分页查询当前状态，记录类型 `sa_map_piece` 为 64 B：`offset/length/file_offset/backend_unit_id:u64@0/8/16/24, backing/protection/sharing/split_policy:u32@32/36/40/44, owner_state:u32@48, flags:u32@52, retained_charge:u64@56`。owner_state：0 LIVE、1 HOLE、2 QUARANTINED；flags bit0 RETAINED_CHARGE_VALID，其他输出零；未VALID时retained_charge不能使用。HOLE 不授予旧 handle 对其中新 owner 的操作权限。

## 6. RT、维护和持久辅助结构

| 类型 | 精确字段顺序及偏移 | 大小 |
| --- | --- | --- |
| `sa_work_budget` | header@0；max_objects/max_pages/max_steps:u64@8/16/24 | 32 B |
| `sa_work_result` | header@0；objects/pages/steps:u64@8/16/24；flags/reserved:u32@32/36 | 40 B |
| `sa_rt_config` | header@0；scan_budget/flags:u32@8/12；total_budget:u64@16；reserved:u64[3]@24 | 48 B |
| `sa_rt_class_spec` | class_index/flags:u32@0/4；slot_size/slot_count/alignment:u64@8/16/24 | 32 B |
| `sa_persist_result` | header@0；requested_stage/completed_stage:u32@8/12；length:u64@16；flags/reserved:u32@24/28 | 32 B |
| `sa_root_entry` | root_id:u64@0；reference:sa_ref@8 | 48 B |
| `sa_object_info` | handle:sa_phandle@0；state/root_references:u32@64/68；reserved:u64@72 | 80 B |

work_budget 每个维度是上限，0 表示该类工作禁用；max_steps=0 为无工作查询，不进行 OS 操作。work_result flags bit0 MORE_HINT 表示可能还有工作，不保证精确剩余数；普通维护的步骤界限不保证 OS 调用的墙钟耗时。

RT config scan_budget=0表示采用arena已求值的1102，其余值直接指定K，最终K必须>0；这与options中1102显式0非法的规则不同。flags bit0 REQUIRE_LOCK；total_budget 为该 pool 全部 payload+metadata+alignment 成本硬上限，0 无效。class_index 必须连续 0..class_count−1；flags bit0 PAGE_CLASS，其他零。普通类 slot_size/alignment 均 >=16、alignment 为二次幂、slot_size 是 alignment 倍数；PAGE_CLASS 还要求 size/alignment 为 P 倍数。slot_count>0。run 按已知 class_index 常数时间寻址，不按任意 size 扫描类表。

prepare的participant数组元素是已登记`sa_ctx*`，无重复且必须包含owner。输出buffer的值记录为`sa_rt_binding_entry`，**无header**：ctx原生指针@0；binding原生指针@W；participant_index:u32@2W；reserved:u32@2W+4；总大小64位24B、32位16B，alignment8。reserved输出零，输出顺序与participant数组相同；版本由buffer头控制，不要求逐项初始化输出头。

任何 RTBinding 存在时对应 ctx detach 返回 BUSY。pool destroy 要求 quiescent、没有 Allocated/Returned 待处理状态，然后撤销全部 bindings。首版不在存活 pool 上导出 owner 迁移函数；需要迁移时在 quiescence 清空、销毁并重新 prepare。

pool destroy的BUSY预检失败保留原pool/bindings可用；空pool通过检查后撤销bindings/用户索引，将既有RP/RC和范围记录转交arena retire链，逻辑销毁返回OK、旧pool/bindings失效。OS回收由普通maintain延后推进，错误保留retired/quarantined记录，不撤销已完成销毁，也不让调用方重试旧pool handle。

persist stage：1 WRITEBACK_SUBMITTED、2 BACKEND_SYNC、3 FILE_BARRIER、4 DEVICE_BARRIER。请求哪个阶段就必须达到该阶段才能 OK；不支持阶段返回 UNSUPPORTED，不悄悄降低。阶段 4 只有后端经验证提供相应设备屏障时可用，不代表所有硬件必遵守断电合同。结果记录实际达到的阶段；COMMIT_UNKNOWN 不得伪装阶段完成。

sa_persist_result的length为请求长度，不是“已成功写回的字节前缀”；completed_stage仅报告**整个请求范围**已经确认达到的阶段，0表示没有可确认的完整阶段。flags/reserved本版输出零。部分成功不能据此推导未报告字节的耐久性。

持久格式profile1的durability_contract_id固定为1，其metadata commit、dirty/clean发布和checkpoint最低要求FILE_BARRIER(3)。后端可以按验证合同提升实际阶段，不能降到1/2；不支持至少3的后端拒绝该持久模式创建/打开。应用显式payload persist可以请求1..4，但只有至少达到本store所需阶段后再发布root，才能声称该payload具备相应恢复保证；allocator不能证明应用是否执行完整payload持久协议。VOLATILE不支付这些成本。

## 7. 能力、统计和默认值查询

所有查询使用 `sa_property` 48 B：`key:u32@0, category:u32@4, value:u64@8, used:u64@16, reserved:u64@24, peak:u64@32, flags:u32@40, state:u32@44`。非配额项的 used/reserved/peak 为零。

category：1 CAPABILITY、2 STATISTIC、3 LIMIT、4 CONFIG、5 ABI_INFO；flags bit0 VALID、bit1 SAMPLED、bit2 QUIESCENT_EXACT、bit3 SATURATED，未 VALID 的数值不得使用。累积统计及其汇总采用u64饱和而非回绕，达到UINT64_MAX时标SATURATED且不得同时宣称QUIESCENT_EXACT；该规则不适用于必须拒绝回绕的身份、配额和地址算术。能力 state：0 UNAVAILABLE、1 IMPLEMENTED_UNVERIFIED、2 VERIFIED；统计/配置 state=0。未执行的平台实验不能报告 VERIFIED。

| capability key | value 含义 |
| --- | --- |
| 1～6 | page_size、allocation_granularity、file_offset_granularity、ordinary_release_granularity、max_arena_capacity、actual_capacity，均 B |
| 7 | address_bits |
| 8～18 | reserve、enable、discard、decommit、prefault、lock、protect、fixed_replace、file_map、page_splittable、RT，value=0/1且结合state |
| 19 | 最大支持 persist stage |
| 20 | source：0 owned VM、1 borrowed buffer |
| 21 | 模式：0/1/2 |
| 22 | health：按sa_error枚举 |
| 23 | native_pointer_bits |
| 24 | required_durable_stage，profile1持久实例为3，VOLATILE为0 |

ordinary_release_granularity仅在后端确实具有适用于该查询范围的统一粒度时非零；value=0且VALID表示“取决于具体mapping/backend unit”，须通过map_query读取真实边界，不能将0误读为任意字节对齐。无arena的platform_query提供P/G/位宽/容量上限和平台潜在能力；actual_capacity、source、mode、health等实例项不设VALID，具体文件系统和来源能力仍由创建及arena_query验证。

statistic keys 2001～2020 按 [S-01 StatsBlock](metadata-layout.md)顺序固定为：allocation_events、free_events、requested_bytes_total、collected_remote_events、live_slot_bytes、live_large_bytes、private_span_bytes、empty_span_bytes、metadata_live_bytes、metadata_reserved_bytes、broker_free_bytes、largest_compatible_free_bytes、mapped_bytes、ordinary_views、fine_views、backend_charge、resident_sample、pending_count、quarantined_bytes、ctx_count。2021=requested_live_bytes，2022=exact_internal_waste；未启用精确小对象追踪时后二者不设置 VALID。

LIMIT/CONFIG key 使用第4节编号。返回实际已解析的值，不返回 AUTO。查询 flags：bit0 REQUIRE_QUIESCENT_EXACT；调用方必须已经建立 quiescence，库不隐式暂停所有线程。未设置时输出允许采样并逐项标记。

持久实例的整个R从创建起属于METADATA，1002的used=R；其中未commit的空白仍不是payload或broker FREE。后续R内切块/启用不再增加metadata地址额度，实际backing启用计入1001及相应统计。metadata_live_bytes、R内已启用字节与R地址总额不能混用，具体成本见配置和持久格式。

## 8. 完整函数目录

以下是文档签名，**不是可编译头文件**。所有函数统一返回 `sa_status`。表内 `const`/指针和参数顺序为后续头文件输入；`e` 均为最后一个可空 `sa_error*`。除输出 buffer 页式合同外，失败时新指针/handle 置空；已有对象/输入句柄不修改。输入和输出存储不得与库将修改的对象、映射范围或彼此重叠。

除e、零count数组、零capacity的buffer.data以及明确接受NULL的对象参数外，表中指针参数均必填；VOLATILE create也传入kind=NONE的sa_native_file，不能以NULL代替必填结构。所有输出调用者存储必须可写且满足对应alignment。property完整数组按key递增输出，不重复；未知key的未来输出由调用方跳过，不能作为未知输入选项重新传回。

### 8.1 实例和控制面

| operation | 函数及参数（均含末尾 e） | 输出/限制 |
| --- | --- | --- |
| 1 | `sa_abi_query(u32 requested_version, sa_buffer *out, sa_error *e)` | ABI_INFO property：key1支持版本、2指针位数、3构建ABI标识；不创建arena |
| 2 | `sa_arena_create(const sa_arena_config *cfg, const sa_option *options, u32 option_count, const sa_native_file *file, sa_arena **out, sa_cleanup_token *cleanup, sa_error *e)` | file为NONE或有效专用文件 |
| 3 | `sa_arena_init_in_place(void *base, size_t length, const sa_arena_config *cfg, const sa_option *options, u32 option_count, sa_arena **out, sa_cleanup_token *cleanup, sa_error *e)` | length必须与cfg.capacity一致 |
| 4 | `sa_arena_open(const sa_native_file *file, const sa_arena_config *cfg, const sa_option *options, u32 option_count, sa_arena **out, sa_cleanup_token *cleanup, sa_error *e)` | cfg.mode必须匹配格式，不改布局 |
| 5 | `sa_arena_destroy(sa_arena *arena, sa_cleanup_token *cleanup, sa_error *e)` | VOLATILE且全部资源已退出，否则BUSY |
| 6 | `sa_arena_discard(sa_arena *arena, sa_cleanup_token *cleanup, sa_error *e)` | 明确整体丢弃，仅VOLATILE、quiescent |
| 7 | `sa_arena_close(sa_arena *arena, sa_cleanup_token *cleanup, sa_error *e)` | 持久正常close，允许live持久对象 |
| 8 | `sa_arena_abort_close(sa_arena *arena, sa_cleanup_token *cleanup, sa_error *e)` | 持久故障/放弃会话；不产生clean承诺 |
| 9 | `sa_arena_query(sa_arena *arena, u32 category, u32 flags, sa_buffer *out, sa_error *e)` | property完整数组，不分页 |
| 10 | 保留，不导出函数 | 首版不支持运行期额度变更 |
| 11 | `sa_arena_maintain(sa_arena *arena, const sa_work_budget *budget, sa_work_result *out, sa_error *e)` | 不等待睡眠owner；可普通OS慢路径 |
| 12 | `sa_thread_attach(sa_arena *arena, sa_ctx **out, sa_error *e)` | 当前线程；累计token耗尽LIMIT |
| 13 | `sa_thread_detach(sa_ctx *ctx, sa_error *e)` | 当前线程；有RT binding则BUSY |
| 14 | `sa_thread_flush(sa_ctx *ctx, const sa_work_budget *budget, sa_work_result *out, sa_error *e)` | 不隐式扫描全部ctx |
| 15 | `sa_cleanup_retry(sa_cleanup_token *cleanup, sa_error *e)` | 仅推进明确可重试清理；成功清空，失败保留剩余责任 |
| 16 | `sa_platform_query(sa_buffer *out, sa_error *e)` | CAPABILITY property；无arena、无外堆，提供create/init所需页几何 |

ABI query 的构建 ABI 标识：1 Windows x64、2 Windows ARM64、3 Linux x64、4 Linux ARM64、5 Android ARMv7、6 Android x86、7 macOS x64、8 macOS ARM64、9 Android ARM64、10 Android x64；平台/工具链实际支持列表仍以发布验证表为准。标识不是持久格式版本。

### 8.2 普通对象

| operation | 普通签名 | context 版本 |
| --- | --- | --- |
| 20 | `sa_malloc(sa_arena *arena, size_t n, void **out, sa_error *e)` | `sa_malloc_ctx(sa_ctx *ctx, size_t n, void **out, sa_error *e)` |
| 21 | `sa_calloc(sa_arena *arena, size_t count, size_t size, void **out, sa_error *e)` | `sa_calloc_ctx(sa_ctx *ctx, size_t count, size_t size, void **out, sa_error *e)` |
| 22 | `sa_realloc(sa_arena *arena, void *ptr, size_t n, void **out, sa_error *e)` | `sa_realloc_ctx(sa_ctx *ctx, void *ptr, size_t n, void **out, sa_error *e)` |
| 23 | `sa_free(sa_arena *arena, void *ptr, sa_error *e)` | `sa_free_ctx(sa_ctx *ctx, void *ptr, sa_error *e)` |
| 24 | `sa_aligned_alloc(sa_arena *arena, size_t alignment, size_t n, void **out, sa_error *e)` | `sa_aligned_alloc_ctx(sa_ctx *ctx, size_t alignment, size_t n, void **out, sa_error *e)` |
| 25 | `sa_usable_size(sa_arena *arena, const void *ptr, size_t *out, sa_error *e)` | `sa_usable_size_ctx(sa_ctx *ctx, const void *ptr, size_t *out, sa_error *e)` |

ctx版本仅替换第一个arena参数，余下参数类型、顺序及返回合同相同，不存在可变参数函数。无ctx入口使用实例预建shared heap；ctx入口只能由其登记线程使用。同一ctx不能并发调用任何入口。

零长分配、free(NULL)、realloc失败保留旧对象和请求长度追踪规则遵循API契约。合法 VOLATILE free不因metadata/queue不足失败。普通对象入口不能在持久实例内隐式创建持久对象，错误模式返回UNSUPPORTED。

### 8.3 页与映射

| operation | 函数参数（含末尾 e） |
| --- | --- |
| 30 | `sa_page_alloc(sa_arena *arena, const sa_page_request *req, sa_page_lease *out_lease, void **out_base, sa_error *e)` |
| 31 | `sa_page_free(sa_arena *arena, const sa_page_lease *lease, sa_error *e)` |
| 32 | `sa_map_estimate(sa_arena *arena, const sa_map_request *req, sa_buffer *out, sa_error *e)` |
| 33 | `sa_mmap(sa_arena *arena, const sa_map_request *req, sa_map_result *out, void **out_base, sa_buffer *report, sa_error *e)` |
| 34 | `sa_munmap(sa_arena *arena, const sa_mapping *map, const sa_range_request *range, sa_map_result *out, sa_buffer *report, sa_error *e)` |
| 35 | `sa_map_replace(sa_arena *arena, const sa_mapping *map, const sa_range_request *range, const sa_map_request *replacement, sa_map_result *out, sa_buffer *report, sa_error *e)` |
| 36 | `sa_mremap(sa_arena *arena, const sa_mapping *map, const sa_resize_request *req, sa_map_result *out, void **out_base, sa_buffer *report, sa_error *e)` |
| 37 | `sa_mprotect(sa_arena *arena, const sa_range_handle *range_handle, const sa_range_request *range, sa_map_result *out, sa_buffer *report, sa_error *e)` |
| 38 | `sa_map_discard(sa_arena *arena, const sa_range_handle *range_handle, const sa_range_request *range, sa_map_result *out, sa_buffer *report, sa_error *e)` |
| 39 | `sa_map_persist(sa_arena *arena, const sa_mapping *map, const sa_range_request *range, u32 stage, sa_persist_result *out, sa_error *e)` |
| 40 | `sa_map_query(sa_arena *arena, const sa_range_handle *range_handle, sa_cursor *cursor, sa_buffer *out, sa_error *e)` |

页/映射入口已经由broker同步，不增加没有快路径价值的 `*_ctx` 同名组，也不借用某个ctx作为共享锁。page_free只接受完整lease，合法逻辑归还不依赖临时metadata；物理回收可延后。borrowed lease的mprotect/discard按能力拒绝，不能改变调用方backing。

map_estimate输出property，category=4，keys3001地址字节、3002新增metadata保守字节、3003backend步骤、3004fine views、3005ordinary views、3006native引用数、3007retained charge、3008report所需条目上界；它不锁定报价、不改变地址所有权。

### 8.4 RT

| operation | 函数参数（含末尾 e） |
| --- | --- |
| 50 | `sa_rt_prepare(sa_arena *arena, sa_ctx *owner, const sa_rt_config *cfg, const sa_rt_class_spec *classes, u32 class_count, sa_ctx *const *participants, u32 participant_count, sa_rt_pool **out_pool, sa_buffer *out_bindings, sa_error *e)` |
| 51 | `sa_rt_try_alloc(sa_rt_binding *binding, u32 class_index, void **out, sa_error *e)` |
| 52 | `sa_rt_free(sa_rt_binding *binding, void *ptr, sa_error *e)` |
| 53 | `sa_rt_maintain(sa_rt_binding *binding, sa_work_result *out, sa_error *e)` |
| 54 | `sa_rt_pool_destroy(sa_arena *arena, sa_rt_pool *pool, sa_error *e)` |

prepare先检查bindings输出容量是否至少participant_count；不足无副作用。bindings归库所有且在arena内，输出数组只是指针记录副本。alloc/maintain只允许owner binding；free允许任何本pool有效binding，调用者必须是该binding绑定的线程。free(NULL)=OK，无ctx fallback不存在。

try_alloc每次执行至多prepare的K次扫描再取指定类freelist；未取得块返回WOULD_BLOCK，不查询全pool判断真满。rt_maintain的objects=本次接收归还数量、steps=状态检查数量、pages=0；不能输出隐式全池统计。合法free一次release-store后不再访问slot/descriptor，不能附加聚合统计原子或日志写入。

### 8.5 持久对象、根与恢复辅助

| operation | 函数参数（含末尾 e） |
| --- | --- |
| 60 | `sa_palloc(sa_arena *arena, size_t n, size_t alignment, u32 flags, sa_phandle *out_handle, void **out_ptr, sa_error *e)` |
| 61 | `sa_pfree(sa_arena *arena, const sa_ref *ref, sa_error *e)` |
| 62 | `sa_ref_resolve(sa_arena *arena, const sa_ref *ref, size_t access_length, void **out_ptr, sa_error *e)` |
| 63 | `sa_allocation_query(sa_arena *arena, const sa_ref *ref, sa_object_info *out, sa_error *e)` |
| 64 | `sa_persist(sa_arena *arena, const sa_ref *ref, size_t length, u32 stage, sa_persist_result *out, sa_error *e)` |
| 65 | `sa_root_get(sa_arena *arena, u64 root_id, sa_ref *out, sa_error *e)` |
| 66 | `sa_root_set(sa_arena *arena, u64 root_id, const sa_ref *ref, sa_error *e)` |
| 67 | `sa_allocation_enumerate(sa_arena *arena, sa_cursor *cursor, sa_buffer *out, sa_error *e)` |
| 68 | `sa_root_enumerate(sa_arena *arena, sa_cursor *cursor, sa_buffer *out, sa_error *e)` |
| 69 | `sa_checkpoint(sa_arena *arena, sa_error *e)` |

palloc n>0；alignment为0表示16，否则为>=16的二次幂；flags首版必须0。pfree只接受delta=0的非空ref；root直接引用仍存在时BUSY。ref_resolve/query/persist不会pin用户业务读者，应用仍需协调free与读取。

root_id范围为0..ROOT_LIMIT−1，超出返回INVALID_ARGUMENT；ROOT_LIMIT=0没有可用根槽。allocation_id范围为1..PERSISTENT_ALLOCATION_LIMIT，ID=0仅用于空ref。sa_persist length必须>0；sa_ref_resolve access_length可以为0但ref仍需指向对象内有效delta，不能借零长度解析one-past。

allocation_enumerate输出sa_phandle，只枚举LIVE对象；root_enumerate输出非空sa_root_entry。allocation_query输出sa_object_info，state：0 FREE、1 LIVE、2 RETIRED_ID；同ID最后一代已经free但尚未复用时可返回FREE帮助判定未知提交，当前代次与ref不同返回INVALID_HANDLE。从未使用的ID返回NOT_FOUND。root_references是allocator根直接引用数，不能证明间接引用不存在。

FREE/RETIRED_ID查询仍输出UUID/ID/generation，offset/requested_length/usable_length和root_references均为0；不能把已释放记录的旧地址作为可访问指针。sa_object_info是固定80B值结构，没有struct_size头，函数要求调用方提供完整对象存储；这与sa_error等版本化输出区别明确。

COMMIT_UNKNOWN后必须停止该实例写入，通过abort-close/open恢复后查询ref。成功返回前崩溃的palloc可能没有调用方已知ref，由allocation_enumerate发现；库不能自动把未在root表出现的对象当垃圾。

## 9. 错误、重入和保留边界

1. 参数/版本/输出容量校验先于副作用；允许对有效输入诊断WRONG_ARENA，不承诺探测任意非法地址而不崩溃。
2. 普通失败不输出新对象，realloc旧对象保持有效。PARTIAL_FAILURE和COMMIT_UNKNOWN是明确例外，分别使用完整片段报告与WRITE_FAULTED恢复合同，不能盲重试。
3. 不设置隐式errno，不分配错误字符串，不启动日志I/O。首版ABI不导出callback注册函数，本版错误通过status/e和显式查询处理。保留REENTRANT状态以便未来受控非RTcallback扩展；没有现行callback就不产生用户回调重入。
4. create/open错误返回前应释放其取得的runtime/native资源；未完成且可重试的清理由cleanup token显式移交。不能把已经写入的专用持久文件伪装成“从未触及”；任何不确定写入按相应I/O状态报告，后续只能open/诊断或明确离线重建。
5. 不提供首版通用provider import、不提供进程级替换、不提供跨进程并发allocator、不提供隐式后台线程。能力字段可以拒绝功能，不能把缺能力静默变成另一种语义。
6. 每个输入/输出buffer生命周期覆盖整个调用。同步API结束后不保存调用者config/options/报告数组指针；RT prepare保留的是库自己的区内binding/class记录，不借用调用者临时数组。

## 10. ABI 冻结门槛

文档完成与发布冻结是两个不同状态。正式头文件生成、导出版本及兼容承诺需要以下证据：

- C17与C++20分别编译，MSVC/clang-cl、GCC/Clang、Apple Clang、Android NDK逐项核对sizeof/alignof/offsetof；32位指针布局单独验收。
- 所有候选status、flag、property key、mode、operation编号通过重复/范围检查；函数目录与导出符号逐一对应，没有未定义例外。
- config默认版本1与S02一致，最小16MiB的默认路径通过S01预算和真实自举验证。
- 32位u64文件offset/容量参数不截断，错误结构不依赖64位lock-free原子；RT实际代码无隐藏锁/CAS循环/OS调用。
- report容量不足无用户范围副作用，PARTIAL_FAILURE总能完整输出；cursor对并发修改返回STALE_CURSOR，不能把不同revision拼成一个快照。
- v1头文件/库的大小兼容、未知字段、未知flags、输出短buffer、调用约定、动态/静态链接和跨模块回收测试完成。
- 不兼容平台实测结果先修订本文/ADR再发布；未通过能力不能被标成VERIFIED。

本文提供确定的实现草案，不替代上述尚未执行的实验，也不意味着已经存在可链接的SuperAlloc库。
