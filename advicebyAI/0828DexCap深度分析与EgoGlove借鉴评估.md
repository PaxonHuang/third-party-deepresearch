# DexCap 深度分析与 EgoGlove 借鉴评估

> 日期：2026-08-28。依据：`repo/DexCap`（j96w/DexCap 克隆，单 commit "202408 hardware updates"）全代码审读 + `paper/2403.07788v2DexCap.md` 论文蒸馏。

---

## 一、TL;DR

1. **DexCap 与 EgoGlove 是互补关系，不是竞争关系**——DexCap 是"背包采集站"形态（手套只是其一个子系统），EgoGlove 是"穿戴手部智能层"；DexCap 的手套用的是 Rokoko 闭源 IMU/EMF 手套，恰恰没有自己做传感器硬件。
2. **不必基于它二次开发**：它的代码是"一次性科研工程"（硬编码、Windows 锁定、无时间同步、T265 已停产、STEP2/3 与 202408 新数据格式尚未适配），作为代码库 fork 价值低；作为**架构蓝图和教训清单**价值很高。
3. **最值得取的精华**：① 手腕 6DoF 用"手背上装 VIO 相机"解决——这正是 EgoGlove Pro 预留 EGO Camera 接口的学术验证（DexCap 用 T265 证明：IMU 漂移问题应靠 SLAM 锚定解决，而不是靠更多 IMU）；② Rokoko→掌系→世界系→机器人系的**四层坐标系变换链**（`redis_glove_server.py` / `visualizer.py:384-399` / `utils.py:132-173`）；③ 指尖 IK 重定向 + 首帧相对位姿 `R_delta_init`；④ robomimic HDF5 数据格式 + 点云 Diffusion Policy 管线；⑤ Human-in-the-loop 残差校正机制。
4. **必须避开的糟粕**：无硬件时间同步（串行轮询+Redis 无时间戳，是它数据质量被诟病的根源，EgoGlove 已把它列为 P0）、人工小键盘校正 SLAM 漂移、外参硬编码（且存在 0.76 vs 0.076 的复制粘贴 bug）、Windows/Rokoko 平台锁定、仅匹配 4 指尖无朝向约束的粗 IK。
5. **兼容建议**：不 fork，但**EgoGlove 的 Robot Action Layer 输出格式应直接对齐 DexCap/robomimic HDF5 约定**（obs/actions/dones/states 结构），因为它已是灵巧手 IL 数据事实格式之一；retargeting 环节复用 `repo/dex-retargeting`（DexPilot 矢量优化版，比 DexCap 自研 PyBullet IK 更先进）而非 DexCap 的实现。

---

## 二、DexCap 是什么：架构与硬件

### 2.1 系统形态
"相机背心 + 背包"：胸口挂 RealSense L515（RGBD 观测）+ 3 台 RealSense T265（1 胸口定位、左右手背各 1 台做腕部 6DoF SLAM），手指用 Rokoko Smartgloves（每手 21 关节 position+quaternion，闭源解算），Intel NUC（Windows）+ 充电宝 + Wi-Fi 路由器背在包里，续航约 40 分钟，整机 ~1.8kg，BOM ≈ $3k–5k。2024-08 更新版弃用已停产的 T265，改用 HTC Vive Tracker + SteamVR Lighthouse 做全局定位，且要求佩戴 VR 头显维持 OpenXR session。

### 2.2 三步流水线
- **STEP1 采集**（背包 NUC）：Rokoko Studio → UDP:14551 → `redis_glove_server.py`（洗数据入 Redis）；`data_recording.py` 串行轮询 4 条 RealSense 流 + Redis，逐帧落盘 jpg/png/pose txt。后处理全靠人工小键盘：SLAM 漂移校正（`replay_human_traj_vis.py --calib`）、偏置估计（`calculate_offset_vis_calib.py`）、桌面系对齐（`transform_to_robot_table.py`）、轨迹剪辑（`demo_clipping_3d.py`）。
- **STEP2 建库**：`pybullet_ik_bimanual.compute_IK` 把 4 个指尖（拇指/食/中/环，小指丢弃）位置解成 LEAP Hand 16 关节；FK 点云拼入观测；SVD 剔桌面、下采样 10k 点；按 `action_gap=5` 写 robomimic HDF5。动作 = 2×(3 平移+4 四元数+16 关节) = 46 维。
- **STEP3 训练**：内置 robomimic fork，`DiffusionPolicyUNetDex`（UNet Diffusion Policy，PointNet/Perceiver 点云编码器，DDIM 10 步推理），观测 horizon 3 / 动作 horizon 10 / 预测 horizon 20。

### 2.3 核心算法思想（公式层）
- **指尖位置 IK 重定向**：只匹配 4 指尖相对腕的位置 `p_finger = p_tip - p_wrist`（腕系内对齐），调用 PyBullet `calculateInverseKinematics`，restPoses 防自碰，输出加 `+=π`/镜像等经验映射。**不是** DexPilot 的矢量空间优化，也没有指尖朝向约束。
- **臂/腕重定向**：首帧相对位姿 `R_delta_init = C ∘ pose0ᵀ`，之后 `goal_ori = R_delta_init ∘ (R_hand ∘ C)`——把世界系漂移问题转化为"相对首帧增量"。
- **点云稳定化**：RGB-D → 点云，统一变换到世界系（主 SLAM 相机首帧），**消除了人躯干移动造成的视角变化**——这是它 in-the-wild 数据比图像输入强 47% vs ~0% 的根本原因（Table II）。
- **机器人手 FK 点云拼入观测**：消人手/机手外观 gap，DP-point 系比 DP-img-mask 平均高约 20 个百分点。
- **Human-in-the-loop 残差校正**：`a'_t = (p_{t+1} ⊕ α·Δp^H_t, J_{t+1} + β·ΔJ^H_t)`，脚踏在残差/全遥操作两模式间切换，校正数据等概率混入微调（类似 IWR）。

### 2.4 关键实验数字
- 30 分钟人类示教数据 → 单手 pick-and-place 平均成功率最高 72%（DP-perc），全程无机器人参与采集，吞吐是遥操作的 3 倍。
- 图像输入 in-the-wild 直接崩溃（~0%），点云世界系稳定化后 47%；30 条人工校正把 Scissor cutting 从 0% 提到 20%。
- 官方承认的三大局限：续航 40 分钟、指尖 IK 无法弥合形貌差异（弹钢琴类任务失败）、**无力觉**（其论文 Future work 就是加触觉织物——与 EgoGlove 的 HKVT 路线撞车）。

---

## 三、与 EgoGlove 的关系判定

### 3.1 互补，且互补得非常具体
| 维度 | DexCap | EgoGlove |
|---|---|---|
| 形态 | 背包/背心采集站 | 穿戴手部智能层 |
| 手指真值 | Rokoko 闭源手套（购买） | 自研 7–11 IMU + HKVT 力传感（开源） |
| 全局 6DoF | T265/Vive Tracker（强项） | IMU 漂移（弱项，无相机/磁锚） |
| 场景观测 | 胸口 RGBD→点云（强项） | 无（Pro 预留 Depth 接口） |
| 下游 | 自带 DEXIL 训练管线（robomimic HDF5） | 数据标准双表示层（管线未通） |
| 商业角色 | 学术开源，无商业化 | 产品化/BP 化 |

**没有竞争**：DexCap 手指数据靠外购 Rokoko，恰是 EgoGlove 想替代的环节；DexCap 的全局定位和场景观测，恰是 EgoGlove 的短板。一个采数据站厂商完全可以用 EgoGlove Pro 替换 DexCap 的 Rokoko 部分——这就是互补。

### 3.2 有必要兼容吗？——有必要，且只兼容"数据格式"
- **对齐 robomimic HDF5**：`data/demo_i/{obs/{agentview_image,pointcloud,robot0_eef_pos/quat,robot0_eef_hand},actions,dones,rewards,states}` 已是 DexCap/DexIL 及一众 IL 工作的事实格式。EgoGlove 的 Robot Action Layer 落盘时按此结构输出，甲方/下游零成本接入。
- **对齐 20Hz 动作约定**：DexCap 60Hz 采、降 20Hz 训练（匹配机器人控制频率）；EgoGlove 采集端高帧率 + 数据集层 20Hz 下采样应成为默认约定。
- **不必兼容**它的采集端（Windows/Rokoko/T265）——没有技术复用价值，且 T265 已停产、Vive 方案要求戴头显。

### 3.3 有必要基于它二次开发吗？——不必
fork 的成本 > 收益：STEP1 锁 Windows+Rokoko 闭源流，代码里全是硬编码（T265 序列号、L515 内参、45° 相机安装角、URDF 指尖偏移），外参还有 0.76/0.076 数值不一致 bug，STEP2/STEP3 尚未适配 202408 新数据布局，且全链无时间同步。EgoGlove 自有固件/总线/链路远比它干净。**正确姿势是"抄架构、抄公式、抄格式"，不抄代码**——个别工具脚本（如指尖 IK 的 restPoses 处理、点云剔除桌面 SVD 流程）可作参考实现单点移植。

---

## 四、取其精华（逐条映射到 EgoGlove 路线图）

1. **"手背相机做腕部 6DoF"= EgoGlove P0 缺口的现成答案**。DexCap 用 T265 证明：腕部全局姿态不该靠 IMU 链条硬扛，而该靠一个 VIO 相机锚定。EgoGlove Pro 的 EGO Camera 接口正好承接（T265 停产，可用带 SLAM 的追踪模组或手机级 VIO/ARKit 方案替代）。这同时回应了 0828 报告里"无全局姿态锚"的 P0-2。
2. **四层坐标系变换链**（Rokoko→掌系→腕系→世界系→机器人系）：EgoGlove 做 MANO/Robot Action Layer 时按同样的分层设计，每层变换独立可校准、可单测，避免 DexCap 式的全局硬编码。
3. **首帧相对位姿 `R_delta_init` + 桌面系一键对齐**：低成本消世界系不一致，`transform_to_robot_table.py` 的"10 秒对齐"交互思想值得产品化。
4. **指尖 IK + restPoses 防自碰**：EgoGlove 接 dex-retargeting 前可先用此法快速出演示；但正式管线应上 `repo/dex-retargeting`（DexPilot 矢量优化，含指尖朝向），因为 DexCap 自研版只匹配 4 指尖位置、丢小指、左手靠镜像 hack。
5. **数据格式对齐 robomimic HDF5 + 20Hz 动作下采样**（见 3.2）。
6. **点云世界系稳定化思想**：若 Pro 加 Depth 采集，观测统一到世界系、机器人手 FK 点云拼入观测，是已被实验证明大幅提升策略成功率的设计。
7. **Human-in-the-loop 残差校正**：EgoGlove 的低延迟穿戴特性天然适合做 DexCap 式校正通道（甚至比背包更轻），可作为 Pro 线的差异化功能："采集 + 校正"双模式。
8. **教训反哺工程**：DexCap 无时间同步导致的数据质量上限问题，正是 EgoGlove P0-3（硬同步）的立项依据——竞品的短板清单就是我们的卖点清单。

## 五、去其糟粕（禁止清单）

1. **无硬件时间戳/软触发**：串行轮询 + 无时间戳 Redis → 多模态几十 ms 级错位。EgoGlove 必须走 DRDY/strobe 硬同步。
2. **人工键盘校正漂移/硬编码外参**：把标定做成产品化流程（自动化、有验收标准），不要小键盘手调。
3. **平台锁定**（Windows、闭源手套、停产硬件、必须戴 VR 头显）：EgoGlove 全线 Linux + 自研 + 可替换模组。
4. **粗 IK 重定向当正式管线**：4 指尖位置匹配 + 经验 offset/镜像 hack，只够演示。
5. **一次性科研代码风格**：magic number 满天飞、STEP 间数据格式漂移。EgoGlove 的四级真实性标注 + 验收测试纪律应保持。

## 六、给甲方的定位话术（一句话）

DexCap 验证了"穿戴手部动捕 + 全局锚 + 场景点云 → 灵巧手策略"这条路是通的（30 分钟数据 72% 成功率），但它用的是 $1000+ 的闭源 Rokoko 手套和已停产的 T265——**EgoGlove Pro 的目标就是以开源、低成本、带触觉预留的姿态替换并超越其手部子系统，同时输出 DexCap 兼容的数据格式直通下游 IL 管线。**

---
来源：`repo/DexCap`（`redis_glove_server.py`、`data_recording.py`、`hyperparameters.py`、`STEP2_build_dataset/pybullet_ik_bimanual.py`、`dataset_utils.py`、`utils.py`、`STEP3_train_policy/robomimic/algo/diffusion_policy.py`）；论文蒸馏 `paper/2403.07788v2DexCap.md`；相关：0828 手部动捕路线盘点（P0 清单）。
