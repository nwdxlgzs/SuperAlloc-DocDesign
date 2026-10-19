# 平台实验导航与执行顺序

状态：全部运行实验未执行。本文件是实验索引；具体输入、步骤、oracle、故障与证据由[平台验证剧本](platform-validation.md)维护，避免两份步骤出现不同判据。功能验收统一对应[验证矩阵](validation.md)，构建与设备runner见[构建规格](build-ci.md)。

## 1. 依赖与验证范围

| 顺序 / 实验 | 主要问题 | 阻塞任务与验收 |
| --- | --- | --- |
| 1 / PV-01 | 连续reserve、4TiB稀疏、P/G、Android32溢出 | I-01、I-04；V-01～V-03 |
| 2 / PV-05 | 原生原子、实际布局、4KiB/16KiB ABI | I-03、I-08；V-03、V-17～V-19、V-27、V-29 |
| 3 / PV-02、PV-03、PV-04、PV-06 | placeholder、邻居访问、普通/可拆、commit回收、POSIX保留 | I-02、I-06、I-09；V-06～V-12 |
| 4 / PV-07、PV-08 | stable file ID、lease、文件屏障及磁盘满 | I-10A；V-21、V-24、V-26 |
| 5 / PV-09、PV-10 | 格式scanner、WAL/checkpoint中断与重建资源 | I-10、I-11、I-12；V-22～V-26、V-36～V-39 |

原子模型、整数检查与字段算术可以先于实机VM实验独立推进；交叉编译不代表平台运行通过。首轮spike使用专用临时文件，标记实验格式，不能直接生成宣称长期兼容的生产store。

## 2. 两个跨层补充场景

**初始化与最终清理失败（V-35）**：reserve成功后强制bootstrap启用失败，再让释放reservation失败；另让destroy在最后control island释放阶段失败。验证caller cleanup token接管唯一责任，旧arena不可继续调用，token不需要外堆。仅重试确认仍归库的资源，不能盲重试可能已关闭并复用的FD。若控制岛已建立，检查剩余记录始终在区内并且控制岛最后清理。

**RT与页驻留（V-17～V-19）**：记录prepare提交、触页和所请求锁页；注入锁页失败必须拒绝prepare。owner暂停，其他登记线程仍可合法free；恢复后建立应用HB，检查完整扫描轮接收。run路径审计无OS/动态分配/锁/CAS循环；状态扫描步数和实际延迟分别记录，不能以预触页代替永不缺页保证。RT teardown逻辑释放后物理回收仍计费直到确认完成。

## 3. 证据和失败处理

统一使用平台剧本的PASS、FAIL、UNSUPPORTED、RESOURCE_LIMIT、NOT_RUN分类。RESOURCE_LIMIT可能证明失败处理正确，但不证明设备成功保留4TiB；缺设备为NOT_RUN。记录OS build/kernel、CPU/ABI、编译器/SDK/NDK、真实P/G、权限、FS/设备、输入配置、native调用序列、原生返回、期望与实际。

地址实验只在已独占reservation内固定替换；先小规模再大范围，4TiB只reserve与稀疏触达。持续访问的测试线程仅访问未操作邻居；目标范围由应用同步停用。磁盘满用隔离限额测试环境，不耗尽用户主卷。创建路径及目录项持久性由调用方完成，库file_ref API只保证文件内容屏障；平台namespace实验验证集成边界，不增加首版路径API。

kill保留OS缓存，只能证明进程中断恢复。torn/reorder模型只证明在指定故障模型中的协议；真实断电仅针对已记录设备/FS/屏障链路形成结论。对应能力在证据不足时保持IMPLEMENTED_UNVERIFIED或UNAVAILABLE，不标VERIFIED。

如果平台无法满足要求，先验证同一arena内、费用可核算的替代后端；仍不满足则不发布该能力并记录影响范围。不能通过静默缩容、外堆回退、临时解除邻居或RT queue-full错误改变已确认契约。ABI/格式候选可先完整写出，正式冻结仍等待这些证据。
