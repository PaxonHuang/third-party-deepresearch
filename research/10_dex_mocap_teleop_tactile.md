# EchoGlove 灵巧操作生态研究综述（MoCap / 遥操作 / 触觉 / 数据契约）

> **Date**: 2026-08-16（首版）· 2026-08-16（源码深研更新：全 14 仓 git clone + 4 组并行子代理源码蒸馏）
> **Status**: 全 paper/ 目录 10 篇论文 + 14 个源码仓深度研究（`third-party-deepresearch/repo/`）；新增 4 份源码蒸馏文档
> **真实性标注**: ✅ 已实现/源码验证 · 🟡 工程可实现 · 🔬 需研发验证 · 🌌 长期方向
> **依据**: `paper/` 全部 10 篇 + `repo/` 源码定点核对；映射对象为 EchoGlove 主线（L1 手套信号链 / Hand Token / L2 双手 / L3 视觉 / Robot Action Layer / 触觉未来）
> **关联**: [[01_signal_chain]] [[02_imu_fusion_madgwick]] [[03_hand_token_protocol]] [[09_fk_mano_dual_rep]]

---

## 0. 全景表（10 篇论文 × 主线相关度）

| # | 论文 | 本质 | 主线维度 | 相关度 |
|---|------|------|----------|--------|
| P1 | **HumanEgo** `2605.24934` | 30min 第一视角视频→零样本机器人策略 | 表示层（ICT）/ 学习 | 🔴 极高 |
| P2 | **DexCap** `2403.07788` | 便携手部 MoCap + 点云 Diffusion IL | MoCap 系统 / 重定向 / 学习 | 🔴 极高 |
| P3 | **FSGlove** `2509.21242` | 16-IMU 48-DoF 手套 + MANO 可微标定 | 手套硬件 / 标定 | 🔴 极高 |
| P4 | **DOGlove** `21_016023` | $600 力反馈手套，21-DoF MoCap + 5-DoF 力反馈 | 手套硬件 / 力反馈 / 遥操作 | 🔴 高 |
| P5 | **AnyTeleop** `2307.04577` | 通用视觉遥操作，学习无关重定向 | 遥操作 / 重定向 | 🔴 高 |
| P6 | **LucidGloves**（源码） | $10-100 DIY 霍尔手套，社区协议 | 手套硬件 / 线协议 | 🟡 中高 |
| P7 | **OSMO** `2512.08920` | 12-taxel 磁触觉手套，人机同手套 | 触觉 / 跨形态 | 🔴 高 |
| P8 | **AnySkin** `2409.08276` | 自粘磁触觉皮肤，跨实例一致 | 触觉 / 可替换性 | 🟡 中高 |
| P9 | **ReSkin** `2111.00071` | 磁触觉皮肤基座，ML 解码 + SSL 适配 | 触觉 / 标定 | 🟡 中 |
| P10 | **FlexiTac** `2604.28156` | $30 压阻阵列 + 3D visuo-tactile 融合 | 触觉 / 数据融合 | 🟡 中 |
| P11 | **Dexbotic** `2510.23511` | 开源 VLA 工具箱 | 学习 / 数据格式 | 🟢 低（借用） |

> 另：**Wuji Glove**（舞肌科技，商业 SDK，`wuji-sdk` 已入库，docs.wuji.tech 提供官方 MCP 端点）——商业对照标杆。

---

## 1. 手套传感方案对比（硬件主线：取哪条路）

这是对本项目最直接的决策输入。五种传感路径对比：

| 方案 | 代表 | 成本 | DoF/精度 | 优点 | 致命短板 |
|------|------|------|----------|------|----------|
| **flex 弯曲**（现状） | EchoGlove ADC1 | 最低 | ~5-11 | 廉价、标定简单（NVS） | 温漂、磨损、只能测弯曲 ✅现状 |
| **霍尔** | LucidGloves | $10-100 | 5-10 | 最便宜 DIY、线性、社区生态 | 磁干扰、需磁铁/霍尔对位 |
| **旋转编码器** | DOGlove | ~$600 | 21-DoF, ±7.2°→±1° | 精确、刚性、可力反馈 | 机械结构复杂、重（550g）、每指编码器 |
| **每指节 IMU** | FSGlove | $426 | 48-DoF, ≤2.7° | 高 DoF、不挡手 | IMU 漂移 3-10°/30min、需每日标定 |
| **单腕 IMU** | EchoGlove LSM6DSV16X | 最低 | 3 euler+3 gyro | 已选定、零额外硬件 | 无指节 DoF（当前 IMU=0，Madgwick 已 host 验证 🟡） |

**工程结论**：
1. **L1 保持 flex + 单腕 IMU 是性价比正确的起点**——SLR 是"粗粒度姿态分类"，不需要指节级 DoF。
2. **LucidGloves 是 flex 的最强替代候选**（若 flex 温漂成为问题）：线协议 `Alpha` 已有完整 C++ 编码 + 社区驱动，迁移成本低。且其"5 弯曲 + 可选 5 splay"拓扑与本项目 5-flex 结构同构。
3. **升级路径清晰**：flex/霍尔（粗）→ 编码器或每指 IMU（精）→ + 触觉（接触力）。FSGlove 证明每指节 IMU 可行但需解决漂移；DOGlove 证明机械精度但贵。

---

## 2. 数据/通信协议契约（表示层：Hand Token 演进的弹药）

### 2.1 Hand Token v1（本项目，79B）vs 外部契约

| 契约 | 维度 | 内容 | 强弱 |
|------|------|------|------|
| Hand Token v1 | 79B 固定帧 | flex×5 + euler×3 + gyro×3（绝对量） | ✅ 紧凑、边缘友好；❌ 无手↔物相对几何 |
| **ICT**（HumanEgo） | 20 维/实体（源码） | `[TypeID(1) \| pose_in_ref(9) \| hand_in_entity(9) \| flag(1)]`（SE(3) 展平 9 维 = 归一化平移 3 + 6D 旋转；手 flag=grasp，物 flag=-1） | ✅ 视点/形体/环境不变、变长实体；❌ 需 SLAM+手部追踪 |
| Alpha（LucidGloves） | ASCII 行 | `A..E 弯曲 + F/G 摇杆 + P 扳机 + 手势字母 + splay` | ✅ 人类可读、调试友好；❌ 冗长、无校验 |
| Dexdata（Dexbotic） | video+jsonl | 多视角图像 + 机器人状态 + 文本提示 | ✅ 省存储；❌ 面向机器人臂数据集 |

### 2.2 ICT 定义（HumanEgo 源码验证 `utils/ict.py:build_ict()`，2026-08-16 深研）

> ⚠️ **Paper↔Source 差异（2026-08-16 记录，以源码为准）**：论文 §3.3 写 29 维 `[τ | REF T_E | E T_LH | E T_RH | g]`（含**双手**相对位姿）；**开源实现是 20 维单手 object-centric**（`utils/ict.py:ICT_DIM_SINGLE_HAND = 20`，仅一个 `hand_in_entity`）。

$$
\mathrm{ICT}_k = [\underbrace{\mathrm{TypeID}}_{1} \| \underbrace{\mathrm{pose\_in\_ref}}_{9} \| \underbrace{\mathrm{hand\_in\_entity}}_{9} \| \underbrace{\mathrm{flag}}_{1}] \in \mathbb{R}^{20}
$$

- TypeID：`0=PAD, 1=HAND_L, 2=HAND_R, 3=OBJ_ANCHOR, 4=OBJ_OTHER`（`max_ict=8`，pad 支持变长物体）。
- `pose_in_ref(9)` = 实体在共享参考帧位姿 = `[normalized_pos(3), o6d(6)]`（归一化平移 + 6D 旋转表示）；`hand_in_entity(9)` = **单只手**在实体局部帧的相对位姿；`flag`：手 = grasp 标量（拇指-食指距离二值化），物 = `-1.0` 哨兵。
- **关键属性（源码确认）**：相对锚定使 `hand_in_entity` 直接反映"接近/抓取/搬运"操作状态；消融证明它是跨形态迁移的主因（raw RGB 7.5% → +ICT 85%）。
- 代码结构：`utils/ict.py`（token 构造）+ `preprocess/DatasetGen.py`（Kinematic State Machine "Latch & Propagate" 物体锁定）+ `training/FlowMatchingModel.py`（MiniPointNet 点云注入）。

### 2.3 数据契约候选（对 M3 上链 / 数据采集的启示）

1. **action = next-state**（DexCap）：`a_t = s_{t+1}`，即动作标签定义为下一时刻完整状态——遥操作数据集的最简形式。
2. **残差修正数据**（DexCap）：`a'_t = (p_{t+1} ⊕ α·Δp_t^H, J_{t+1} + β·ΔJ_t^H)`，β<0.1，修正数据与原数据各 50% 采样微调——人机协同数据增强。
3. **分位数归一化**（OSMO）：`x' = clip((x - x_0.02)/(x_0.98 - x_0.02), -1.5, 1.5)`——触觉/状态通道的标准预处理。
4. **离散动作 token**（Dexbotic）：每动作维分 256 bins 进 LLM tokenizer——Hand Token v2 若走 VLA 侧的接口候选。

---

## 3. 标定（信号链主线：从 flex NVS 到可微 MANO）

| 方法 | 出处 | 公式/过程 | 精度 | 对本项目 |
|------|------|-----------|------|----------|
| flex NVS 标定 | 现状 | 0..1 归一化 | ✅ | 保留 |
| **旋转编码器线性映射** | DOGlove | $\alpha = \frac{V_{ADC}}{V_{CC}}·360°$ | ±7.2°→±1°（外部高精编码器校正表） | 若上编码器硬件 |
| **DiffHCal**（MANO 可微标定） | FSGlove | 3 参考姿态最小二乘解 A,C_i；接触集形状误差 $E_{shape}=\sum_{c_k \in C}\|v_j(θ,β)-v_k(θ,β)\|$ | ≤2.7°关节；≤3.6mm 网格 | **research/09 直接对接**：把 flex/IMU 原始量送进 MANO 可微优化，统一解关节+形状+安装误差 |
| 穿戴重标定 | FSGlove | 手套拉伸→每日重标定 | — | 运维约束 |

**踩坑记录（DiffHCal 关键点）**：IMU 与指节链接 frame 未对齐 → 引入校正旋转 C_i（`R_M·C_i = A·R_i`）；需 ≥2 参考姿态 + 最小二乘解 A 和 C_i；建议 3 姿态（rest / x 轴转腕 / y 轴转腕）。接触集用 MANO 顶点索引（拇指尖 744、食指尖 320）。

---

## 4. 重定向与遥操作（Robot Action Layer 主线）

### 4.1 指尖重定向：跨 4 篇收敛的共识

DOGlove（FK 指尖→Mink IK）、DexCap（指尖 IK，LEAP 弃小指）、OSMO（MuJoCo IK 直接用人类指尖）、HumanEgo（手→虚拟平行夹爪）**全部以指尖位置为迁移原语**。最优形式化（AnyTeleop）：

$$
\min_{q_t} \sum_{i=0}^{N} \|\alpha v_t^i - f_i(q_t)\|^2 + \beta\|q_t - q_{t-1}\|^2 \quad \text{s.t.} \quad q_l \le q_t \le q_u
$$

- $v_t^i$：人类手第 i 个关键点向量；$f_i(q_t)$：机器人手 FK；α：尺寸缩放；β：时序平滑权重；关节限位约束。
- **验证**：AnyTeleop 用该优化在真实任务 8/10 超越为特定硬件调制的 Telekinesis，且**学习无关**（仅需 URDF 即可换机器人）——对 EchoGlove 的"通用遥操作层"是可直接借鉴的实现。
- **源码验证（2026-08-16）**：`repo/AnyDexRetarget/` (`fce83d1`) — `AdaptiveOptimizerAnalytical` 融合 TipDirVec（捏合精度）和 FullHandVec（张开姿态），per-finger pinch alpha 平滑过渡，~2ms/帧（NLopt SLSQP + 手写解析梯度）。支持 13 个机器人手（Shadow/Wuji/Allegro/Inspire/Ability/Leap/SVH/LinkerHand L21/L20/ROHand/Unitree Dex5/Sharpa/Gaia Hand20）。**EgoGlove 集成路径**：Hand Token v2 FK 派生 21 点 → MediaPipe 21-point 格式 → AnyDexRetarget 输入 → 机器人关节角。新增机器人仅需 URDF + YAML config。

### 4.2 手→夹爪/机械手转换（HumanEgo，防退化关键）

- 位置：$p_{ee} = \frac{1}{2}(p_{thumb\,tip}+p_{index\,tip})$（拇指+食指指尖中点）。
- 姿态：**MCP 关节构建 Gram-Schmidt 框架**（而非指尖）——避免捏合时指尖收敛导致的退化；$\mathbf{x}_{ee}=\widehat{p_{iMCP}-p_{tMCP}}$（夹爪开合轴），$\mathbf{y}_{ee}=\tilde{y}-(\hat{\tilde{y}}^\top\mathbf{x}_{ee})\mathbf{x}_{ee}$（腕→MCP 中点），$\mathbf{z}_{ee}=\mathbf{x}_{ee}\times\mathbf{y}_{ee}$。
- 开合：$g=\mathrm{clip}(\frac{\|p_{thumb\,tip}-p_{index\,tip}\|-d_{min}}{d_{max}-d_{min}},0,1)$，部署时二值化。
- 平滑管线：置信度掩码(<0.8 丢弃) → 间隙插值(SLER) → Savitzky-Golay(窗 21, 阶 2) → EMA(α=0.15) + Gram-Schmidt 再正交 + 180° 翻转抑制。

### 4.3 腕部 6-DoF（EchoGlove 当前缺失的绝对位姿）

- **DexCap**：EMF 手套（指节）+ T265 SLAM（腕 6-DoF，漂移自校正）+ L515 RGB-D 胸戴（3D 观测）。SLAM 解腕位姿是可穿戴、可长时间、抗遮挡的成熟路线。
- **FSGlove**：IMU 解腕姿态 + 外接光学（Nokov/Vive）解腕平移。
- **HumanEgo**：Aria MPS 闭环节流轨迹（SLAM+回环+全局优化）直接给重力对齐世界系 6-DoF。
- → 对本项目：若要做「手↔物相对几何」级表示，腕部绝对位姿是前置依赖；当前 flex+单 IMU 无法给出（yaw 不可观，见 [[02_imu_fusion_madgwick]]）。

---

## 5. 数据采集与策略学习（L2/L3 学习主线）

### 5.1 生成式策略：diffusion vs flow matching（二选一）

| | Diffusion（DexCap/OSMO/AnySkin） | Flow Matching（HumanEgo） |
|---|---|---|
| 形式 | 去噪 latent 轨迹 | 线性插值 + 速度场回归 |
| 训练损失 | DDPM | $\mathcal{L}_{FM}=\mathbb{E}[\ w_p\|\Delta p\|^2 + w_r\|\Delta r\|^2 + w_g\|\Delta g\|^2\ ],\ x_t=(1-t)x_0+tx_1$ |
| 推理 | 100 去噪步 | 固定步 Euler ODE |
| 权重 | — | $w_p=5,\ w_r=1,\ w_g=10$ |
| 适用 | 任务级、已验证（DP3） | 快、少数据（HumanEgo 92.5%） |

### 5.2 密集辅助损失（HumanEgo，少数据关键）

$$
\mathcal{L} = \mathcal{L}_{FM} + \lambda_{OM}\mathcal{L}_{OM} + \lambda_{2D}\mathcal{L}_{2D} + \lambda_{LC}\mathcal{L}_{LC}
$$

三个辅助头共享编码器，分别做**前向动力学预测**于三种空间：3D 物理空间（object motion：预测物体未来 9-D 位姿轨迹）、2D 视觉空间（visual foresight：预测关键点 2D 投影，权重 w_f=20）、潜空间（latent consistency：预测 K 步后的 ICT）。消融：三者独立各 +17.5/+12.5/+5pp，合计 +25pp；**15min 数据时收益最大**。另有 region attention spotlight（对 2D 锚点高斯聚焦）+ state-noise injection。

### 5.3 多模态观测架构（OSMO/AnySkin）

- **OSMO**：per-modality 编码（2 MLP 处理状态+触觉，DINOv2 冻结编码图像）→ concat 成全局条件 → FiLM 条件 U-Net 去噪。触觉张量 $g \in \mathbb{R}^{3\times2\times5}$（XYZ × 双磁强计差分 × 5 指尖），差分传感减共模噪声。
- **AnySkin**：BAKU——每模态专属编码器（图像 ResNet-18 / 触觉 MLP）→ token 序列 → transformer encoder，action token 输出，action chunking=10。
- **3D-ViTac 融合**（FlexiTac）：视觉转 3D 点云 + 触觉经 FK 提升为 3D 触点（触觉量作 feature channel + modality 指示）→ 统一 3D visuo-tactile 点表示 → 点云扩散策略。**这与 HumanEgo 的点云注入 ICT 是同一哲学的两种实现**。

---

## 6. 触觉与力反馈（硬件未来主线）

### 6.1 触觉传感技术路线对比

| | 磁触觉（OSMO/AnySkin/ReSkin） | 压阻阵列（FlexiTac） | 力反馈执行器（DOGlove） |
|---|---|---|---|
| 成本 | ~$30-80/单元 | ~$30/整块(≤384 taxel) | ~$600(主动) |
| 能力 | 法向+剪切（3 轴） | 法向为主 | 主动力反馈（电机） |
| 形态 | 自粘皮肤/手套 | FPC 薄片(<1mm) | 绳驱机械 |
| 代表硬件 | BMM350×2 + 软磁弹性体 + MuMetal 屏蔽；MLX90393×5 | FPC-Velostat-FPC 三层压合，2mm 电极 | Dynamixel 伺服 + 0.6mm 钢索双向滑轮 |
| 关键坑 | 环境磁场/铁磁物干扰；串扰（MuMetal+差分抑制 57%） | Velostat 非线性、时漂 | 重、速度受限 |

### 6.2 力反馈分层策略（DOGlove，可移植的阈值表）

| 力传感器读数 | 触觉(LRA) | 力反馈(伺服) |
|---|---|---|
| <10 g | ✗ | ✗ |
| 10–50 g | ✓ | ✗ |
| 50–100 g | ✓ | ✓ |
| >100 g | ✗ | ✓ |

- 伺服电流-位置双环：力读数 [0,3000g] 线性映射到伺服 K_P；LRA 波形 ID 56 "Pulsing Sharp 1-100%"。
- **工程洞察**：连续触觉会误导操作者对表面细微特性的感知（滑瓶实验），故 100g 处切断触觉只留力反馈。

### 6.3 可替换性（AnySkin，数据规模化前提）

- 脉冲磁化器固化后磁化（非固化中磁场）→ 信号强、跨实例 std 0.12 vs ReSkin 0.54。
- 细颗粒 MQFP-15-7(25μm) → 防沉降、分布均匀。
- 自对准自粘（免胶水/螺丝）→ 12s 更换、可复用；跨实例策略迁移仅 -13%（ReSkin -43%）。
- 触觉数据要可规模化，**跨实例一致性 > 绝对精度**。

---

## 7. 对 EchoGlove/EgoGlove 主线的选择性吸收（精华/糟粕取舍）

### ✅ 精华（吸收）

| 决策点 | 吸收来源 | 落点 |
|--------|----------|------|
| **Hand Token v2 候选 = ICT 式交互 token**（相对 SE(3)+抓取，视点不变） | HumanEgo P1 | Hand Token v1（绝对 flex+IMU）之上叠加交互层；L3 视觉侧输入格式 |
| **指尖重定向 = 优化式**（学习无关、仅需 URDF） | AnyTeleop P5 / DexCap P2 / DOGlove P4 | Robot Action Layer + research/09 FK→MANO 验证 |
| **少数据训练 = flow matching + 密集辅助损失** | HumanEgo P1 | L2/L3 训练管线；HumanEgo 8min 数据 > ACT 30min 遥操作 |
| **生成式策略架构 = diffusion policy + 分位数归一化** | OSMO P7 / DexCap P2 | L2 动作解码器候选 |
| **数据格式 = video+jsonl（Dexdata）+ action=next-state** | Dexbotic P11 / DexCap P2 | M3 relay 数据管道 |
| **触觉升级路线 = 先 AnySkin 式磁触觉（若上触觉）** | AnySkin P8 / OSMO P7 | 硬件 roadmap；共享人/机手套平台理念 |
| **低成本手套替代 = LucidGloves 霍尔 + Alpha 协议** | LucidGloves P6 | flex 失效时的 drop-in 备选 |
| **可微标定 = DiffHCal（MANO 接触集误差）** | FSGlove P3 | 把 flex/IMU 原始量对接 MANO 做形状感知标定 |

### ❌ 糟粕/风险（丢弃或标注）

| 风险 | 来源 | 处理 |
|------|------|------|
| 每指节 IMU 漂移 3-10°/30min + 每日重标定 | FSGlove P3 | 不用于实时 SLR；仅作 research/02 的漂移边界参考 |
| 纯视觉策略在接触密集任务失败（55.75% vs 触觉 71.69%） | OSMO P7 | 佐证"视觉 L3 需触觉/力补足"；非否定视觉 |
| 视觉遥操作自遮挡丢跟踪（AnyTeleop 失败模式） | AnyTeleop P5 | L3 视觉侧标注限制 |
| 磁触觉对环境铁磁物敏感 | ReSkin P9 / AnySkin P8 | 触觉硬件选型限制条件 |
| 编码器 ±7.2° 未标定误差 | DOGlove P4 | 若上编码器必须配校正表 |
| flex 温漂/磨损 | FSGlove 综述 | 现状已知；霍尔为备选 |
| VLA 工具箱（Dexbotic）单机不适用 | Dexbotic P11 | 仅借格式与 token 化思想，不引入框架 |

---

## 8. 给架构会话的 5 个关键决策点

1. **Hand Token v2 是否引入 ICT 式交互 token**？（表示层最大分歧点：绝对 vs 相对）——HumanEgo 消融是其最强论据，但需腕部 6-DoF + 物体追踪前置。
2. **腕部绝对位姿如何补**？（DexCap T265-SLAM / HumanEgo Aria-SLAM / FSGlove 外接光学）——当前 flex+单 IMU yaw 不可观。
3. **重定向实现选优化式（AnyTeleop）还是 IK 库（Mink/MuJoCo）**？——影响 Robot Action Layer 的代码基座。
4. **L2/L3 动作解码器选 diffusion policy 还是 flow matching**？——配合密集辅助损失。
5. **触觉是否纳入第一代硬件**？纳入的话路线三选（详见 §11.3 buy vs build）：A) **COTS buy**——HKVT-M3A 单点 I2C drop-in（最省事）/ PaXini PX-6AX 阵列（抗杂散磁场但 83.3Hz）；B) **开源 DIY**——AnySkin 式磁触觉（剪切+法向，400Hz，需自制）；C) **压阻**——FlexiTac（廉价法向阵列）。力反馈（DOGlove 主动）标 🌌。

---

## 9. 溯源表（2026-08-16 更新：全 14 仓 git clone）

| 论文/仓库 | arXiv/URL | 本地源码 | Commit |
|-----------|-----------|----------|--------|
| HumanEgo | [2605.24934](https://arxiv.org/pdf/2605.24934) | `repo/HumanEgo/` | `0921b2d` |
| DexCap | [2403.07788](https://arxiv.org/pdf/2403.07788) | `repo/dex-retargeting-main.zip`（相关库） | — |
| FSGlove | [2509.21242](https://arxiv.org/pdf/2509.21242) | `repo/fsglove/` | `a7988f0` |
| DOGlove | [2502.07730](https://arxiv.org/pdf/2502.07730) | `repo/DOGlove-main/` | ZIP（无 git） |
| AnyTeleop/AnyDexRetarget | [2307.04577](https://arxiv.org/pdf/2307.04577) | `repo/AnyDexRetarget/` | `fce83d1` |
| LucidGloves | [wiki](https://github.com/LucidVR/lucidgloves/wiki) | `repo/lucidgloves/` | `76472c7` |
| OpenGloves-Driver | — | `repo/opengloves-driver/` (develop) | `9e1f2fd` |
| OSMO | [2512.08920](https://arxiv.org/pdf/2512.08920) | `repo/osmo_tactile_glove/` | `bfc7328` |
| AnySkin | [2409.08276](https://arxiv.org/pdf/2409.08276) | `repo/anyskin/` | `cb13b5b` |
| ReSkin | [2111.00071](https://arxiv.org/pdf/2111.00071) | `repo/reskin_sensor/` | `b82de2a` |
| FlexiTac | [2604.28156](https://arxiv.org/pdf/2604.28156) | `repo/FlexiTac/`（Awesome-FlexiTac gallery） | `6c5d111` |
| Dexbotic | [2510.23511](https://arxiv.org/pdf/2510.23511) | `repo/dexbotic/` | `6356c98` |
| Wuji Glove (SDK) | [docs](https://docs.wuji.tech) | `repo/wuji-sdk/` | `4f7e8bf` (v2026.8.3) |
| Wuji Glove (CLI) | — | `repo/wuji-cli/` | `76766c8` (v2026.8.3) |
| Wuji Glove (Description) | — | `repo/wuji-description/` | `06e5f14` (v2026.8.14) |
| HKVT-M3A 指尖触觉（航凯） | — | `usermanual/HKVT-M3A_指尖触觉传感器用户手册.md` | Ver.01 |
| PaXini PX-6AX GEN3（帕西尼） | https://www.paxini.com | `usermanual/PaXini_PX-6AX_GEN_3_Series.md` | GEN3 |

**新增源码蒸馏文档**（`paper/` 目录）:

| 文档 | 覆盖源 | 行数 |
|------|--------|------|
| `LucidGloves_DOGlove_driver_research.md` | LucidGloves + OpenGloves-Driver + DOGlove | 912 |
| `WujiGlove_research.md` | Wuji SDK + CLI + Description | 1028 |
| `Teleop_Retarget_FlexiTac_research.md` | AnyDexRetarget + dexbotic + FlexiTac | 647 |
| `Tactile_OSMO_AnySkin_ReSkin_research.md` | OSMO + AnySkin + ReSkin | 364 |

> Wuji docs MCP: `https://docs.wuji.tech/mcp`（已加入 Codex config.toml `mcp_servers.wuji-docs`，下次重启生效）

---

## 10. 源码深研补充（2026-08-16，4 组并行子代理 + 直研）

### 10.1 FSGlove HI229 IMU 串行协议 ✅

FSGlove 手套服务器用 Go 编写，通过串口读取 16 个 HI229 IMU 的原始帧：

```text
HI229 串行帧格式（Chaohe 协议）:
┌──────┬──────┬────────────┬───────────┬───────┬─────┬───────┐
│ Sync1│ Sync2│ PayloadLen│ Payload... │ CRC16 │ ... │ items │
│ 0x5A │ 0xA5 │ U16 LE    │ TLV items │ U16   │     │       │
└──────┴──────┴────────────┴───────────┴───────┴─────┴───────┘

TLV item codes:
  0x90 = ID (IMU node ID)
  0xA0 = AccRaw (I2 × 3, /1000 → g)
  0xB0 = GyrRaw (I2 × 3, /10 → deg/s)
  0xC0 = MagRaw (I2 × 3)
  0xD0 = RotationEul (float × 3)
  0xD1 = RotationQuat (float × 4)
  0xF0 = Pressure
```

- 串口配置: 115200 baud, 5s read timeout, 4096 byte buffer
- gRPC 上行: `IMUPacketResponseRaw` protobuf（id + accel_xyz + gyro_xyz + mag_xyz + quat_wxyz + roll/pitch/yaw）
- 源码: `repo/fsglove/src/hand_server/internal/sensor/hi229/hi229_serial.go`

**与 EgoGlove Hand Token v2 对比**：FSGlove 用 protobuf gRPC 做上行（重量级），EgoGlove 用 79B 固定帧 + ESP-NOW（轻量级）。但 TLV item 编码思路与 Hand Token v2 的 capability-flagged TLV 设计同构。

### 10.2 FSGlove Raw2MANOCalibrator 标定管线 ✅

`repo/fsglove/src/hand_visualiser/modules/hand/calibration.py` 中的标定分三阶段：

1. **全局安装误差**（`_get_global_installation_error`）: 3 参考姿态解 IMU→MANO 坐标系旋转偏差 $C_{global}$（16 个关节各一个）
2. **局部安装误差**（`_get_local_installation_error`）: 沿 FK 链传播，逐关节校正 $C_{local}$（rotvec）
3. **GPR 精修**（`GPRInstallationCalibrator`）: 用高斯过程回归残差精修 16 个关节的安装误差

MANO 参数提取:
```python
# FK 序列: [1,4,7,10,13, 2,5,8,11,14, 3,6,9,12,15] (15 个指节关节, index 0 = 腕)
# parent mapping: self._joint_fa (来自 AxisLayerFK)
# delta_rot_fa_rotvec: rest pose 下 MANO 坐标系与 axis-layer 坐标系差异
rotvec_mano[idx] = (cur_rot.inv() * global_inst_error[idx] * delta_imu[idx] * local_inst_error[idx].inv()).as_rotvec()
```

**与 EgoGlove 对接**：若 Pro 版上指节 IMU，此标定管线可直接参考。当前 Lite 的 flex+单腕 IMU 不需要此级标定，但 DiffHCal 的 MANO 接触集误差概念可用于 flex→MANO 的形状标定（research/09）。

### 10.3 Wuji Glove EMF 定位方案 ✅

`repo/wuji-sdk/` (v2026.8.3) 确认舞肌采用电磁定位（EMF）而非 flex/IMU 弯曲传感：

```
腕部 TX 线圈 → 5 个指尖 RX 线圈 → 每指尖 6-DoF 位姿 → 在线 IK → 21-DoF 关节角
```

- `EmfPose`: `pose.position (xyz, 米)` + `pose.orientation (x,y,z,w)` + `confidence [0,1]`
- `HandJointAngles`: 5 个手指各 4 关节角 (radians) + per-finger confidence
- 拇指 5 DOF (CMC_rot/flex/abd + MCP + IP) + 食指~小指各 4 = **21 DOF**
- URDF: `wuji-description/glove/body/urdf/right.urdf` — 21 revolute joints
- **四元数顺序: w-last** `(x,y,z,w)` — 与 EgoGlove 的 **w-first** `(w,x,y,z)` 相反，互操作时必须转换
- 触觉矩阵: 744 values (24×31)，v2026.7.14 从 768 降为 744（删除坏列）
- IK 并行化: v0.7.0 起跨 CPU 核，`emf_poses_rate_divider` 可降采样

### 10.4 AnyDexRetarget 自适应优化器 ✅

`repo/AnyDexRetarget/anydexretarget/optimizer/analytical_optimizer.py` — 源码验证的公式：

$$\mathcal{L}(\mathbf{q}) = \sum_{i=1}^{N_f} \left[\alpha_i \cdot \mathcal{L}_{\text{TipDirVec},i} + \beta_i \cdot \mathcal{L}_{\text{FullHand},i}\right] + \lambda \|\mathbf{q} - \mathbf{q}_{\text{prev}}\|^2$$

- Pinch alpha: $\alpha_i = \text{clip}(\frac{d_2 - d_i}{d_2 - d_1}, 0, 1)$，$d_1=2.0$cm, $d_2=4.0$cm
- $\alpha=1$ → TipDirVec（捏合精度），$\alpha=0$ → FullHandVec（张开姿态）
- Huber loss $H_\delta$ 对位置和方向分别加权
- NLopt SLSQP + 手写解析梯度 → ~2ms/帧
- 13 个机器人手配置 YAML：Shadow/Wuji/Allegro/Inspire/Ability/Leap/SVH/LinkerHand L21/L20/ROHand/Unitree Dex5/Sharpa/Gaia Hand20

### 10.5 LucidGloves Alpha 协议 + DOGlove 76B UART 帧 ✅

**LucidGloves Alpha 文本协议**（`repo/lucidgloves/src/Encoding/AlphaEncoding.cpp`）:
```
A<curl0>B<curl1>C<curl2>D<curl3>E<curl4>          # 5 弯曲值 0-1023
(J<splay0>K<splay1>L<splay2>M<splay3>N<splay4>)   # 可选 splay
(G<joyX>H<joyY>)                                    # 可选摇杆
(T<trigger>)                                        # 可选扳机
```

**DOGlove 76B UART 帧**（`repo/DOGlove-main/DOGlove-main/glove_mcu.py`）:
```
76 bytes binary: 21 channel × 3 bytes (24-bit encoder raw) + header + 16-bit sum checksum
Encoder: V_raw / V_ref × 360° → ±7.2° raw, 外部高精编码器校正表 → ±1°
```

### 10.6 OSMO Bowie 手套数据通道 ✅

`repo/osmo_tactile_glove/labs/glove2robot/utils/glove_utils.py`:
- 40 个磁场值 (5 taxel × 8 channels) + 20 个四元数分量 (5 IMU × 4)
- USB CDC 设备名: "BowieGlove"
- 线程化 `BowieGloveStream` + `queue.Queue` 异步读取
- 管线: Bowie 手套 → HaMeR 关键点 → Psyonic Ability Hand 重定向 → diffusion policy

## 11. 商用触觉传感器（buy 路线，2026-08-16 新手册）

> 来源：`usermanual/HKVT-M3A_指尖触觉传感器用户手册.md` + `usermanual/PaXini_PX-6AX_GEN_3_Series.md`

### 11.1 HKVT-M3A（苏州航凯，单点 3 轴 I2C 指尖触觉）✅ 规格来源：手册

- **规格**：法向 Fz=15N + 切向 Fx/Fy=10N，精度 2%FS，安全过载 400%；硅胶表面；2.5-3.3V；200Hz 采样。
- **接口**：I2C 从机 **0x0A**（7-bit），400kHz；4P 下翻盖 FPC（AFC42-S04FMA-1H）；自带上拉可直连 MCU。
- **协议**：命令 `0x03` 读 6 字节（XYZ 各 2 字节 signed int16，**小端**）；`0x1A` 写改 I2C 地址（Flash 持久，勿设 0xFF/0x00）。
- **踩坑**：上电校准约 1s；**需周期性清零（零点漂移）**；明示避免强磁场/尖锐物/高温高湿环境。

### 11.2 PaXini PX-6AX GEN3（帕西尼，Hall 阵列多轴触觉）✅ 规格来源：手册

- **原理**：半柔双层——形变层（压力→磁场畸变）+ 刚性传感层（Hall + 算法），**内置 Anti-Stray 抗杂散磁场功能**（算法剥离环境磁场干扰）→ 直接回应 ReSkin/AnySkin/OSMO 的环境磁敏感短板。
- **阵列**：12 型号（Elite/Core/Omega），指尖 25-135 taxel、手掌 9、大 CP 239 taxel，每 taxel 三轴；空间分辨率 1mm；量程 Fz 0~25N / Fxy ±10N；精度 1%FS；最小 0.1N；寿命 10M+ 次。
- ⚠️ **输出频率仅 83.3Hz**（低于 100Hz 实时控制需求）；电流 100-700mA；供电 3-5V。
- **三协议**（上电 CS3/CS2/CS1 引脚选通，运行中不变）：SPI(CPOL=High/CPHA=2Edge/MSB 优先，CLK 串 51Ω) / UART(921600 8N1，请求-响应) / I2C(≤200kHz，4.7k 上拉)。
- **帧格式**：`55 AA | len(2B LE) | dev_addr(=module+1) | 00 | 0xFB(读)/0x79(写) | start_addr(4B LE) | data_len(2B LE) | [data] | LRC`；读响应回 `AA 55`。与项目 `uart_frame.h`（0xAA 0x55 + CRC）帧头同构。
- **寄存器**：0x79 用户区（addr3=校准置 1）；0x7B 应用区（**1008/1009/1010=合力 Fx/Fy/Fz** 各 1B，Fz=值×0.1N；**1038+=逐 taxel Fx/Fy/Fz** 各 1B，单字节：Fx/Fy -128~+127，Fz 0~255）。

### 11.3 buy vs build（对架构决策点⑤的直接输入）

| 维度 | HKVT-M3A | PaXini PX-6AX | 开源 DIY（AnySkin/OSMO） | FlexiTac |
|---|---|---|---|---|
| 轴/规模 | 3 轴单点 | 3 轴阵列(9-239) | 3 轴磁（剪切+法向） | 法向阵列 |
| 采样率 | 200Hz | **83.3Hz** ⚠️ | 400Hz | 100Hz |
| 抗环境磁场 | ❌（明示避免） | ✅ **Anti-Stray** | ❌（环境铁磁敏感） | ✅（压阻不敏感） |
| 接口 | I2C 0x0A | SPI/UART/I2C（需通信板） | 定制 STM32→USB | Arduino→USB |
| 集成成本 | 低（drop-in） | 中高 | 高（自制） | 低 |

**结论**：① 若第一代硬件要"低成本加 1-2 个指尖触觉点"，**HKVT-M3A 是最高性价比 drop-in**——S3 已有 I2C 总线，0x0A 与 LSM6DSV16X(0x6A) 不冲突；② 若需覆盖整个手面做接触定位/纹理，PaXini 阵列是 buy 路线的正解，但 83.3Hz 限制其用于动态控制（分类/接触状态足够）；③ 400Hz 高带宽触觉（滑移/动态）仍需开源 DIY 路线（AnySkin/OSMO）；④ 触觉（接触力）与 LSM6DSV16X（运动）在传感器层正交——决策点⑤的选型弹药在此。
