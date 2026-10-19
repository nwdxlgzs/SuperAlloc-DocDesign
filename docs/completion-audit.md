# 需求覆盖与文档完成审计

审计范围：用户已确认的SuperAlloc架构计划，以及“规划、技术文档、待办、指导全部编写”的文档目标。本阶段**不实现分配器、不生成头文件/构建工程、不执行平台/运行时/性能实验**。完成文档不是证明allocator已正确、已发布或已领先业界。

日期2026-10-11；当前v0.3候选已完成安全回退、状态机及高负载专项交叉审查与静态核验。v0.2核验记录保留在第3节，v0.3增量记录在第5节，任务勾选以[TODO](../TODO.md)为准。ABI、磁盘格式和平台支持承诺尚未经过实现验收或发布冻结。

## 1. 用户要求逐项覆盖

“文档证据”表示当前文件有可判定的规范与失败合同；“实施验收”是未来必须取得的运行证据，均尚未执行。

| ID | 明确要求 | 当前文档证据 | 实施验收 |
| --- | --- | --- | --- |
| R-01 | Windows/Linux/macOS/Android，x64/ARM64及Android32 | [平台](platforms.md)版本/ABI矩阵，[构建](build-ci.md)工具链，[剧本](platform-validation.md)实际P/ABI | I-01～I-03；V-02、V-03、V-27、V-29 |
| R-02 | C++20内部、稳定C ABI；不依赖外部堆/STL动态分配/异常/RTTI | [指导](development-guide.md)编译与依赖约束，[ABI](c-abi-spec.md)完整字段/函数/版本，[元数据](arena-metadata.md)外部固定状态界限 | I-03、I-04、I-14；V-01、V-27、V-29 |
| R-03 | 显式独立函数、多实例、非系统替换 | [API](api-contracts.md)显式arena/ctx入口，[架构](architecture.md)实例独立锁/额度，[接入](integration-guide.md)线程用法 | I-05、I-07；V-05、V-42 |
| R-04 | 单连续地址区域同时对象/页映射，碰撞可控 | [架构](architecture.md)唯一broker、不变量、布局偏好，[操作](operation-protocols.md)Pending及固定地址授权 | I-04、I-06、I-09；V-06～V-10 |
| R-05 | malloc至少16B，页对齐使用真实4KiB/16KiB | [API](api-contracts.md)长度/对齐/0/溢出，[并发](concurrency.md)44类与slab，[平台](platforms.md)运行时页大小 | I-03、I-05、I-06；V-03、V-04、V-12 |
| R-06 | 16MiB～4TiB，32位独立上限 | [预算](metadata-budget.md)自举和稀疏目录，[配置](configuration.md)三容量算例，[平台](platforms.md)PTRDIFF_MAX限制 | I-01、I-04；V-01～V-03、V-30 |
| R-07 | 全部可增长管理结构区域内自举 | [布局](metadata-layout.md)32/64字段，[预算](metadata-budget.md)64KiB初始库存、应急成本，[格式](persistent-format.md)R/scratch/辅助区 | I-04、I-10；V-01、V-08、V-29、V-32、V-39 |
| R-08 | 区间分割合并、固定替换、失败回滚不丢所有权 | [操作](operation-protocols.md)逐路径信用、OS前Pending、失败片段和隔离；[元数据](arena-metadata.md)兼容合并 | I-04、I-09；V-06～V-11、V-34 |
| R-09 | 高性能本地路径、跨线程释放、owner退出接管 | [并发](concurrency.md)release CAS/acquire exchange/live晚扣/无全局快路径计数，[操作](operation-protocols.md)detach/adopt/退休 | I-07；V-13～V-16 |
| R-10 | 普通完整free不因元数据耗尽失败 | [API](api-contracts.md)逻辑free，[操作](operation-protocols.md)既有intrusive retire记录和延后OS清理 | I-05、I-07；V-01、V-08、V-32 |
| R-11 | 缓存/提交/线程/metadata/map/维护分别可控 | [配置](configuration.md)12额度、6策略、immutable求值、三维维护，[ABI](c-abi-spec.md)options/查询/错误资源键 | I-04、I-07；V-16、V-33 |
| R-12 | 自行保留与借用已映射buffer两种初始化 | [平台](platforms.md)能力/权限边界，[API](api-contracts.md)init，[操作](operation-protocols.md)borrowed逻辑划分 | I-05、I-06；V-20、V-42 |
| R-13 | 匿名/文件子区间、SHARED/PRIVATE、保护/回收/刷新 | [API](api-contracts.md)映射族，[平台](platforms.md)offset及能力，[ABI](c-abi-spec.md)请求/签名/保护/阶段 | I-06、I-09；V-09～V-12 |
| R-14 | Windows普通/可拆、逐页替换邻居不解除、view上限 | [平台](platforms.md)placeholder/section粒度，[配置](configuration.md)view预算，[剧本](platform-validation.md)PV-02～PV-06 | I-02、I-09；V-10、V-11、V-34 |
| R-15 | 多页操作可部分失败，输出完整且调用方同步 | [操作](operation-protocols.md)完整preflight及partial结果，[ABI](c-abi-spec.md)caller buffer/STALE_CURSOR/句柄洞身份 | I-06、I-09；V-09、V-10、V-34 |
| R-16 | 受限RT固定准备、有界步骤、任意登记线程free必成功 | [实时](realtime.md)状态机/内存序/K/HB界限，[ABI](c-abi-spec.md)binding与静态class，[操作](operation-protocols.md)准备/逻辑销毁 | I-08；V-17～V-19、V-40 |
| R-17 | 临时、正常重开、元数据崩溃恢复分开 | [持久化](persistence.md)三模式与提交点，[格式](persistent-format.md)CLEAN/DIRTY、WAL与双super状态转换 | I-10～I-12；V-22～V-26、V-38 |
| R-18 | 相对引用、版本、校验、日志/checkpoint、重建锁与索引 | [格式](persistent-format.md)精确LE偏移/CRC64/256B槽/目录/流式scanner/64N辅助，[ABI](c-abi-spec.md)ref/handle | I-10、I-11；V-26、V-36～V-39 |
| R-19 | 元数据成功持久提交；payload由应用persist/业务事务管理 | [持久化](persistence.md)分配/初始化/persist/root流程，[格式](persistent-format.md)stage3及未知结果，[接入](integration-guide.md)数据库协议 | I-11；V-23～V-25、V-42 |
| R-20 | root/分配枚举、孤立对象不自动删除 | [ABI](c-abi-spec.md)root与目录ID/generation/query/enumerate，[持久化](persistence.md)直接root阻止free及应用间接引用 | I-11；V-24、V-25、V-36 |
| R-21 | 每主文件单进程管理、file identity独占锁，外部map会话有效 | [持久化](persistence.md)lease，[格式](persistent-format.md)新空file_ref/open重验，[平台剧本](platform-validation.md)PV-07 | I-10A、I-10；V-21、V-25、V-41 |
| R-22 | destroy忙碌、持久close保留live、关闭与擦除分离 | [API](api-contracts.md)生命周期，[操作](operation-protocols.md)cleanup token/close/abort-close，[运行](operations-guide.md)后续动作 | I-05、I-10；V-28、V-35 |
| R-23 | 语言运行时、引擎、数据库接入 | [集成指南](integration-guide.md)具体生命周期/错误分支，[运行](operations-guide.md)维护和观测 | I-05、I-08～I-11、I-14；V-42 |
| R-24 | PLAN/TODO、架构/API/布局/状态机/内存序/平台/格式/ADR | [导航](README.md)所有文档入口，[PLAN](../PLAN.md)阶段，[TODO](../TODO.md)依赖/输入/交付/验证/完成条件 | 文档D/S；后续I阶段逐项产证据 |
| R-25 | 一步一步推进，多Agent分工与交叉审查 | [开发指导](development-guide.md)归属及修改规则；本审计第2节收敛记录 | 本次三Agent实际交付与整合；非额外运行要求 |
| R-26 | 错误/耗尽、故障注入、弱内存序、断电与kill分开验证 | [验证](validation.md)V-01～V-43，[详细剧本](platform-validation.md)PV-01～PV-10，[构建](build-ci.md)分层 | I-01～I-12，不以文字推导替代实测 |
| R-27 | 低延迟高吞吐、业界基准对比、空间与尾延迟同时报告 | [基准](benchmarking.md)固定竞争对手版本、环境manifest、重复采样、冷/热/RT/持久分组及5%/10%/5%门槛 | I-13～I-15；无已测排名 |
| R-28 | 可执行指导、版本/冻结/发布、后续研究边界 | [构建](build-ci.md)、[运行](operations-guide.md)、[发布](release-policy.md)；[TODO](../TODO.md)F-01～F-06 | I-14发布材料/兼容/许可决策，F项首版外 |

## 2. 交叉审查收敛记录

本轮三Agent分别交付操作协议、C ABI、持久格式与平台剧本，主设计整合配置、规划、验证和指导。已解决的跨层问题包括：

- quota预付与record信用分离，用户请求不占着增长所需最后额度等待；普通free/完整page lease/RT teardown使用已有退休控制记录。
- report空间在用户范围OS修改前足够，partial结果不依赖外部堆；游标不能拼接不同revision，洞复用不扩张旧handle权限。
- revision在Pending开始和最终发布分别推进、预留收口次数；饱和时关闭分页而保留合法释放和清理，不以诊断机制制造新的free失败。
- 控制岛不可访问时由固定caller token持有清理责任，不能在解除控制区之后读取清理链；未确定native close结果不盲重试。
- 持久R整体归metadata、地址used一次预付、实际commit另计；borrowed增长仅改逻辑所有权。
- root/free验证在同一commit序列，checkpoint排空已提交未publish操作；WAL满请求先退出自身admission，避免等待自己结束。
- 双super发布完成后才复用旧WAL/CP，DIRTY也双发布；FREE目录槽保留generation，root不指会话映射。
- ABI使用file_ref，不声称能创建路径/同步目录；caller namespace责任明确。候选字段与发布冻结分开，保留所有实测门槛。

这些是设计审查结果，不是调度模型、编译或OS测试。条件布局、native锁原位大小、Windows粒度、文件屏障和恢复故障模型仍须I阶段证据。

## 3. v0.2文档核验记录

v0.2交付时对该版本工作区执行了静态检查和独立算术复核，历史结果如下；当前v0.3增量结果见第5节。检查使用临时内联脚本，未向项目加入实现、测试脚本或二进制文件。

| 核验项 | 实际结果与范围 |
| --- | --- |
| 文件与导航 | 29份Markdown，非文档文件为0；247处本地文件链接有效。全部docs文档可从docs导航抵达，全部29份文件可从根README抵达 |
| 文档结构 | 代码围栏闭合；无未填写的设计占位；本检查未验证外部网站可用性 |
| 需求与编号 | R-01～R-28、V-01～V-43、PV-01～PV-10、ADR-001～ADR-034连续且无重复定义；ABI状态码0～29、12项额度键及6项策略键核对通过 |
| 任务依赖 | 29个D/S/I任务节点，展开区间依赖后无未知引用、无环；13项文档任务完成，16项I阶段任务及6项F项均未完成 |
| Runtime字段 | 13张顺序字段表、75组字段的32/64位偏移、对齐及末端已核对；分区布局与槽位预算另作交叉审查。不是编译器sizeof或原子lock-free结果 |
| 自举预算 | 19行初始库存偏移连续；L0合计23424/22656/16896B分别对应64位4KiB、64位16KiB、32位4KiB，均容于65536B；应急库存及多view信用公式已复核 |
| C ABI字段 | 24个固定布局及4个原生指针布局变体完成字段算术核对；输出buffer、cleanup token及无header的RT binding值记录同步。正式C/C++头文件编译和ABI测试未执行 |
| 磁盘字段 | 6张序列化头部/记录表共97个字段及填充跨度，覆盖其声明长度且无洞、无重叠；不将runtime结构直接序列化 |
| 持久几何与校验 | 16MiB、64MiB、1GiB、4GiB、4TiB五组R/J/K/N公式复核；16MiB固定保留1966080B、payload14811136B、M_init262144B、checkpoint有效内容73728B；CRC-64/ECMA-182样例得到0x6C40DF5F0B497347 |

文档完成范围为D-01～D-07和S-01～S-06。上述证据说明规范已写出并完成本轮检查；不能据此宣称不存在实现缺陷、已达到性能目标或已支持所有平台。

未执行项明确保留：编译/sizeof/native atomics、任何真实VM与文件实验、并发/RT模型、sanitizer、kill/torn/reorder/断电、benchmark与应用集成运行。不存在这些结果不阻塞本次文档交付，但阻止实现任务和公开性能/兼容结论被标完成。

## 4. 下一阶段边界

本轮文档完成后继续保留完整I阶段待办。用户明确要求开始程序时，先执行I-01～I-03的适用可行性验证和模型，再进入单线程正确性核心；任何实验反例都先更新对应契约/ADR/预算，不能以实现已经写出为由降低已确认要求。

## 5. v0.3安全回退与高负载专项

输入为用户“安全回退、状态机、高并发环境、高吞吐环境的处理设计打磨”要求。三Agent分别细化状态/操作协议、并发/性能、平台/存储，主集成同步API、布局、配置、指导与实施依赖；本次仍为纯文档工作。

| 专项 | 已写入的设计与审查结论 | 后续运行证据 |
| --- | --- | --- |
| 状态与发布 | [状态机](state-machines.md)定义多维状态、唯一执行者、转换守卫和pin；新资源先结算再发布，Pending名额后退；同mapping变更串行避免最终发布失配 | V-44～V-47 |
| 安全回退 | [失败与恢复](failure-recovery.md)区分无效果、可完整恢复、部分生效、未知；终态隔离不占active Pending但保留X/J证据；禁止超时抢占、盲重试native身份和递归回滚 | V-44～V-47、V-52、V-55 |
| 高并发维护 | [并发](concurrency.md)明确release sequence、发布后零访问、全部owned slab环和双游标；中央/退休固定批次独占claim，shared冷类也有维护路径 | V-48～V-50 |
| 高吞吐与过载 | [争用与吞吐](contention-throughput.md)区分发布/接收/复用服务率、积压和公平性；[基准](benchmarking.md)加入独立到达计划、有限测试队列及删失/漏发统计，维护CPU计入成本 | V-50、V-51 |
| 持久并发故障 | persist进入admission并pin，fault与阶段许可有序；确认commit仍完成发布并返回OK，health单独反映故障；CHECKPOINT_HELD只续缺失镜像 | V-53、V-54 |

本轮不增加公开状态/函数、磁盘字段、quota键或每对象全局原子。HeapHeader原未定义flags同宽改为维护cursor，类环和队列根同宽改名；终态X复用active_next、Q extent复用free链字段，均明确互斥生命周期。SlabDesc对齐明确64B，现有64B头/192B槽位预算不变。每次维护的16指针工作区为固定栈成本，不能误称所有清理工作没有代价。

最终静态核验结果：32份Markdown、无非文档文件，306处本地链接有效，导航可达且围栏闭合；30个D/S/I任务无未知依赖或环，14项D/S完成且依赖均已满足，16项I和6项F未完成。V-01～V-55、ADR-001～ADR-040连续无重复定义，公共状态0～29及18个option键保持不变。

重新核对13张runtime字段表/75组字段的32/64位对齐和偏移、6张磁盘字段表/97个字段及填充、19行自举库存、五组持久容量公式与CRC标准向量，结果与v0.2相符。检查的是文档算术，未编译sizeof、未运行调度模型、OS/恢复实验或benchmark。所有V-44～V-55仍为未执行场景；新增ADR记录机制与代价，不构成无缺陷、性能领先或平台已支持的结论。
