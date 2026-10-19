# 平台可行性验证剧本

状态：**全部运行项未执行**。本文规定后续I-01～I-03、I-10A及格式故障模型的输入、步骤、oracle和证据；当前只完成设计，不创建或运行spike代码。能力合同参见[platforms.md](platforms.md)，持久候选参见[persistent-format.md](persistent-format.md)。

## 1. 环境矩阵与结果分级

每次实验记录：OS名称/build、CPU型号/架构、P与allocation granularity、地址宽度、内存/commit额度、文件系统及mount属性、存储设备/驱动、编译器/SDK/NDK、进程权限、测试seed、配置与源版本。没有这些字段的“成功”不能作为支持证据。

| 目标 | 最低维护环境 | 必需执行面 | 当前状态 |
|---|---|---|---|
| Windows x86-64 | Windows 10 1803+ desktop | MSVC和clang-cl构建；现代实机与最低API兼容环境 | 未执行 |
| Windows AArch64 | 支持所选API的Windows ARM设备 | 原生设备，不能只用x64模拟层 | 未执行 |
| Linux x86-64/AArch64 | kernel 5.4+ | GCC/Clang；至少一个4KiB与可获得的其他页几何 | 未执行 |
| macOS x86-64/AArch64 | macOS 11+ | Intel与Apple Silicon真实VM行为 | 未执行 |
| Android ARMv7/arm64 | API 23+，NDK r28+构建 | 32位ARM、64位ARM、4KiB/16KiB；最低API与现代页配置分别覆盖 | 未执行 |
| Android x86/x86-64 | 同项目ABI兼容范围 | 模拟器用于ABI/错误路径补充，不替代ARM验证 | 未执行 |

结果区分：PASS（限定所记环境）、FAIL（违反oracle）、UNSUPPORTED（平台明确缺能力）、RESOURCE_LIMIT（环境额度不足）、NOT_RUN。4TiB保留受环境约束，RESOURCE_LIMIT可以是正确API行为，但不能作为4TiB成功证明。

CI适合结构布局、编译、格式模型、普通syscall回归；模拟器适合API级别和错误路径。物理页大小、缓存一致性、锁页、真实文件系统屏障、存储断电与尾延迟必须有相应实机证据。不得把kill、虚拟机重启或删除缓存等同于物理断电。

## 2. PV-01：reserve、页几何与容量

**步骤**：查询真实P、G、地址宽度和native atomic宽度；在无负载实例分别请求16MiB、32位上限附近、64位4TiB；记录成功范围与保留后的resident/commit，仅读写最初、末尾和稀疏若干页。重复创建多个独立范围后解除其中一个，确认其他范围不变。32位执行C、offset+length、round-up与size_t转换的边界输入。

**Oracle**：请求成功则地址连续、长度精确、不重叠；失败不输出可用arena。清理成功时无残留，清理失败时通过固定caller token交付全部仍可清理的资源责任并保留未知状态诊断，不能声称无残留。稀疏创建不按全域页数建立用户元数据；4TiB保留不导致4TiB提交。32位任何算术溢出先于OS调用被拒绝。P来自运行时而不是编译宏猜测。

**故障**：地址碎片、RLIMIT/commit额度、保留成功后bootstrap失败、第二实例建立失败。记录释放后region查询结果；不能把失败范围留给可见arena handle。

**证据**：逐次地址/长度/native error表、保留前后commit/resident样本、系统region快照、算术边界清单。当前NOT_RUN；阻塞I-04对应平台。

## 3. PV-02：Windows placeholder与partial mapping

**步骤**：保留至少16MiB placeholder；在非64KiB但P对齐位置拆出3P；以`MapViewOfFile3(MEM_REPLACE_PLACEHOLDER)`映射文件中非64KiB但P对齐offset。验证返回地址；按完整view unmap-preserve并coalesce。另建三个各P view，左右线程反复校验各自唯一pattern，中间线程解除中页并换到另一个文件，持续一百万次或明确资源限制停止。

分别测试`VirtualAlloc2` private replacement的P、64KiB、G地址/尺寸矩阵。不得由file-view成功推导private replacement同样合法。ordinary单大view的partial请求应在用户态预检拒绝；不能先撤销整个view再补回两侧。

**Oracle**：可拆模式左右pattern与可访问性持续正确；中间页只显示对应文件数据；解除恢复placeholder、地址仍归reservation。ordinary拒绝不产生OS副作用。每个file view的offset、base、length粒度与能力表一致。

**故障**：split失败、第二/第k个view建立失败、unmap-preserve失败、coalesce失败、文件handle失效、metadata配额不足。已完成步骤逐片记录；只能证明恢复内容时声明rollback，否则PARTIAL_FAILURE/QUARANTINED。

**证据**：每个组合的API参数/返回/error、左右线程错误计数、region查询、view与placeholder数量、故障前后范围账本。当前NOT_RUN；阻塞Windows I-02/I-09和fullVM支持声明。

## 4. PV-03：POSIX保留范围内替换与保护

Linux、macOS、Android分别执行，不以Linux结果替代其他系统。

**步骤**：匿名PROT_NONE保留16MiB；按真实P启用匿名子区；映射文件A/B的P对齐子区间，覆盖仅限自有且已验证FREE范围；对中页执行保护变更、解除并恢复保留态。另一线程反复申请地址空间并监测目标区，不允许在受管理范围制造先munmap后重新抢占的洞。

**Oracle**：左右合法区间不受影响；固定请求不覆盖malloc/metadata；释放后的range仍处于受控保留状态。MAP_FIXED_NOREPLACE不能替换已经存在的PROT_NONE映射，后端必须按其真实语义处理。读取保护测试置于独立子进程，期望fault的访问不能直接使主测试进程失去证据。

**故障**：FD权限、非对齐offset、文件短于范围、mprotect失败、映射数量限制、restore失败。匿名内容被丢弃后不能通过重建零页假装原内容恢复。

**证据**：syscall参数、maps/region快照、子进程退出状态、邻居hash、注入步骤与最终owners。当前NOT_RUN；阻塞各POSIX后端I-02/I-09。

## 5. PV-04：匿名资源回收与Windows section生命周期

**步骤**：建立相同大小的private allocation、SEC_COMMIT pagefile section、SEC_RESERVE section三组；分别提交、预触、撤销部分view、保留另一view、解除最后view、关闭最后section引用。每步采样commit charge、resident与可访问性；在隔离进程内验证`VirtualFree`对已提交SEC_RESERVE页的行为。Linux/macOS/Android分别验证采用的discard/decommit策略对内容与resident的实际合同。

**Oracle**：private decommit与section view解除不能混称。若一部分view解除后commit仍保留，统计必须报告retained；要求立即回收的profile不得冒充已满足。4TiB reserve不得建立4TiB SEC_COMMIT对象。resident下降不是commit退款证明。

**故障**：commit额度、锁页、内存压力、section创建失败、部分视图仍被引用。记录资源不足后是否可继续释放已取得资源。

**证据**：native创建flags、每步handle/view引用数、系统commit计量与取样方法、内容pattern、平台回收能力结论。当前NOT_RUN；阻塞anonymous splittable策略与backend charge统计发布。

## 6. PV-05：4KiB/16KiB、ARMv7原子与自举

**步骤**：同一逻辑配置在4KiB/16KiB设备运行；ARMv7构建检查u64显式8B对齐和pointer/a32原子，反汇编确认热路径未调用libatomic锁。记录S-01每项sizeof/alignof/offsetof；在16MiB实例拦截被测模块外部heap调用，执行创建、首次attach、首次slab、metadata增长、销毁。运行时测试与宿主测试框架自身分配分开计数。

**Oracle**：所有元数据均位于arena；布局不符时明确重算设计而非越界。32位逻辑cap不超PTRDIFF_MAX页对齐上限。RT依赖的load/store必须lock-free，普通路径也不得隐含使用未经验证的128位CAS。持久4KiB store在不兼容16KiB环境打开返回INCOMPATIBLE_GEOMETRY。

**故障**：最低metadata额度、native锁槽超预算、首次ctx不足、metadata-growth第二次故障、不可lock-free目标。合法volatile free继续成功，RT prepare拒绝不支持目标。

**证据**：编译器版本、布局diff、符号/反汇编、外部heap计数、16MiB分账、不同页几何拒绝记录。当前NOT_RUN；阻塞I-03/I-04与RT平台能力。

## 7. PV-06：view预算、文件内容与多arena隔离

**步骤**：ordinary和PAGE_SPLITTABLE分别创建单map达到4096细粒度view、多个map达到16384细粒度view；普通views另计。先调低metadata配额，验证map_estimate能暴露view配额尚可但元数据不足。两个arena映射同一外部SHARED文件，核对共享数据；PRIVATE模式写入后核对原文件未被修改。partial unmap形成洞再由新handle占用，旧handle重试跨洞操作。

**Oracle**：配额是项目策略而非OS硬上限；不得超限后静默降级ordinary。旧handle不能操作洞中新owner。文件offset、长度、权限和有效handle生命周期符合API；两个arena的统计和故障独立。

**故障**：在view第1、中间、最后一步失败；文件外部截断/介质错误放入子进程以观察OS异常；新handle关闭原FD后继续访问验证库自有引用。

**证据**：预估与实际元数据/view/handle账、pattern文件hash、各阶段rollback结果、原生异常分类。当前NOT_RUN；阻塞I-09。

## 8. PV-07：主文件身份与owner lease

**步骤**：在每个拟支持的本地FS创建实验文件；通过原路径、相对/绝对别名、硬链接（平台支持时）、重复handle，同进程及第二进程尝试打开同一store。POSIX采用独立open-file-description的flock；Windows采用LockFileEx。持锁时关闭另一个非owner FD、终止owner进程、再竞争。测试不同主文件的多arena与同一外部payload文件共享。

**Oracle**：同一文件第二allocator返回BUSY，不能只靠路径比较；关闭非owner引用不意外释放owner lease；进程退出后可以重新取得。外部payload共享不被错误当作主文件重复owner。PID重用不触发自动抢锁。

**故障**：锁API不支持、权限不足、文件被替换、锁取得后初始化失败、全局登记后控制区撤销、文件系统不提供所需语义。失败必须撤销进程内登记，不留悬垂节点；不支持锁合同则不提供durable模式。

**证据**：native file-ID、打开/关闭时间序列、锁owner进程事件、返回状态、初始化失败后的重试。仅通过被测本地FS可加入支持名单。NFS/SMB和不明用户态FS首版直接UNSUPPORTED，不以“通常锁可用”放行。当前NOT_RUN；阻塞I-10A/持久创建。

## 9. PV-08：文件flush、命名空间与磁盘满

**实验后端候选**：Linux/Android `msync(MS_SYNC)`加适当`fdatasync/fsync`；macOS显式映射同步加文件同步，要求更强设备屏障时验证F_FULLFSYNC；Windows FlushViewOfFile之后FlushFileBuffers。库只接收file_ref/FD，创建路径和父目录同步由调用方负责，namespace级合同作为应用部署实验单独验证。候选调用组合不是已证明的统一断电保证。

**步骤**：在隔离实验目录创建固定大小文件，写pattern A并完成合同屏障，写pattern B但停在不同同步阶段；另进程读回，进程kill后重开，随后用受控虚拟故障后端模拟写回乱序/撕裂。分别测试文件扩展、稀疏洞写入、目录项创建与rename。磁盘满使用受限测试卷，不耗尽用户主磁盘。

**Oracle**：分别报告WRITEBACK_SUBMITTED、BACKEND_SYNC、FILE_BARRIER、DEVICE_BARRIER；profile1的持久提交至少需要FILE_BARRIER。调用方命名空间同步另列，不混成库能力；要求库保证namespace的请求必须UNSUPPORTED。truncate读零不代表物理空间预留；sparse写失败可能成为映射fault，不能虚构所有错误都能转成status。

**故障**：msync/flush/fsync每步失败、ENOSPC、只读卷、设备断开、外部截断、父目录无同步能力。主文件结果不明进入WRITE_FAULTED，不继续复用；外部session失败不污染其他arena。

**证据**：FS/device/driver清单、每次调用次序和native结果、kill后文件hash、调用方创建路径的目录存在性、模拟持久集合与期望集合。真实断电是额外设备级测试，只在专用环境和明确安排下执行；当前未执行，也不要求以破坏用户设备的方式获得证据。当前NOT_RUN；阻塞durability_contract_id支持声明。

## 10. PV-09：格式scanner与提交故障模型

此项首先可在纯模型/CI上实施，之后在每个平台的实验文件后端复核。

**步骤**：按persistent-format v1生成16MiB最小store；构造ALLOC、FREE、ROOT_SET序列。对redo写、redo屏障、commit A/B、commit屏障、runtime apply、API返回每个边界分别中止。再对checkpoint复制、CP屏障、super A发布、super B镜像和旧WAL复用各边界中止。切分粒度涵盖1B、8B、512B、4KiB、P与G；重排只允许发生在未被确认屏障约束的写之间。

**Oracle**：恢复接受的事务形成已验证的连续提交前缀；所有成功返回的元数据事务存在，未返回的事务允许存在或不存在但不能半应用。有效commit配损坏redo为CORRUPT；一个损坏commit副本不要求丢弃另一有效副本；中间缺口后有有效commit必须拒绝。双super完成前不重用旧日志，因此任一可选旧链仍有其所需数据。

**故障输入**：CRC不同位置翻转、合法CRC但offset越界/重叠、N/Q/长度溢出、未知features、最大generation/LSN、重复slot、root指FREE/旧代次、当前epoch中间缺口、旧epoch残留。故意制造CRC碰撞不作为安全认证能力，解析边界仍必须成立。

**证据**：seed、完整操作trace、每个写对应的持久字节集合、恢复目录/roots、预期status、反例最小化文件。当前NOT_RUN；阻塞I-11与磁盘格式发布。

## 11. PV-10：持久容量、重建与数据发布

**步骤**：创建16MiB默认N1024/Q64实例，报告R/J/双K和payload精确分账；分配满N后再申请，释放并复用同ID验证generation增加；推进超过J/256次事务触发checkpoint换epoch。open显式降低1002应因几何不匹配拒绝，降低可覆盖的backend/view预算或构造稀疏span布局另验证实际重建预估；分别在4KiB/16KiB与32位容量不兼容环境打开副本。使用“palloc→写payload→persist→root_set”发布，并在每步终止。

**Oracle**：N、payload、runtime、日志是独立额度；不因payload仍空就越过目录上限。R全部METADATA且1002used=R，未启用R单元不被用户map占用，也不要求eager commit；R内增长不重复1002收费。FREE tombstone代次进入checkpoint；checkpoint不能丢失尚未runtime apply的已提交事务。root仅指主文件持久对象，不指session map；恢复产生未root的LIVE对象时枚举可见，不能自动回收。显式data persist不自动成为应用事务。

**故障**：checkpoint满、R不足、N耗尽、root直接引用对象pfree、COMMIT_UNKNOWN后重试、session尚存时close、DIRTY clean store重开。对应行为分别LIMIT/METADATA_LIMIT/BUSY/WRITE_FAULTED/拒绝clean重开，不能静默退回VOLATILE。

**证据**：精确region表、slot代次历史、checkpoint代次、WAL epoch、metadata实际高水位、root/payload校验与恢复枚举。当前NOT_RUN；阻塞I-10/I-11验收。

## 12. 证据归档与能力发布门槛

每个PV结果记录固定字段：PV ID、项目revision、环境指纹、输入配置、步骤序号、观测值、oracle比较、status、证据文件路径、未覆盖项、对应能力/实施任务。原始trace保留，不只交付截图或“通过”摘要。

能力注册表只对经过同类环境验证的组合标VERIFIED。实现存在但未验证标IMPLEMENTED_UNVERIFIED；API/权限/FS明确缺失标UNAVAILABLE。不把某个Linux内核结果扩展成所有Android，也不把普通文件同步测试扩展成所有SSD断电保证。

发布阻断关系：PV-01/05约束基础平台；PV-02/03/04/06约束映射与回收；PV-07/08约束durable后端可用性；PV-09/10约束格式/恢复发布。缺少平台设备时可以继续独立文档和模型工作，但该平台结果保持NOT_RUN，不能删去测试目标来伪装完成。
