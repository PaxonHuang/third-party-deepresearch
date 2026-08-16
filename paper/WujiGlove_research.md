# Wuji Glove (舞肌科技) — SDK / CLI / Description 研究蒸馏

> **Date**: 2026-08-16
> **Researcher**: Codex (deepresearch-distill)
> **Subject**: Wuji Technology (舞肌科技) commercial glove + dexterous hand ecosystem
> **Purpose**: EgoGlove 项目竞品/参考分析 — 对标 Wuji Glove 数据采集与遥操作能力
> **Reality convention**: ✅ 已实现 (repo 证据) / 🟡 工程可实现 / 🔬 需研发验证 / 🌌 长期方向
> **未核实声明**: 标注"待人工核对"

---

## 0. 来源与可追溯性

| Repo | Path | Commit | Release |
|------|------|--------|---------|
| wuji-sdk | `/home/EchoGloveHugeProjects/third-party-deepresearch/repo/wuji-sdk` | `4f7e8bf` | v2026.8.3 |
| wuji-cli | `/home/EchoGloveHugeProjects/third-party-deepresearch/repo/wuji-cli` | `76766c8` | v2026.8.3 |
| wuji-description | `/home/EchoGloveHugeProjects/third-party-deepresearch/repo/wuji-description` | `06e5f14` | v2026.8.14 |

**在线文档** (部分内容未在 repo 中出现，引用时标"待人工核对"):
- 主站: https://docs.wuji.tech/docs/zh/wuji-glove/latest/
- 触觉数据参考: https://docs.wuji.tech/docs/zh/wuji-glove/latest/sdk-data-reference/tactile/
- CLI 文档: https://docs.wuji.tech/docs/en/wuji-cli/latest/
- Description 文档: https://docs.wuji.tech/docs/en/wuji-description/latest/

**产品家族** (从 README + CHANGELOG 推断):
- **Wuji Glove** — 数据手套，EMF + 触觉 + IMU 传感，手部追踪
- **Wuji Hand** (一代) — 灵巧手，20 关节，USB 连接
- **Wuji Hand 2** (Beta 1 / Beta 2) — 灵巧手，20 关节 + 指尖触觉，Ethernet 通信
- **Wuji Studio** — 配套 GUI 软件（多客户端共享设备）

---

## 1. 核心算法/公式

### 1.1 EMF (Electromagnetic Field) 定位 — ✅ 已实现

Wuji Glove 的核心手部追踪**不依赖 flex 弯曲传感器**，而是采用电磁定位方案：

- **腕部 EMF 发射器 (TX)**: 佩戴在手腕的电磁发射线圈（URDF: `emf_tx_base` link）
- **指尖 EMF 接收器 (RX)**: 每个指尖一个接收线圈（URDF: `thumb_rx_coil` ... `pinky_rx_coil`）
- **原理**: 5 个指尖 RX 线圈感应腕部 TX 产生的电磁场，解算每个指尖相对腕部的 6DoF 位姿
- **输出**: `EmfPoseArray` — 5 个 `EmfPose`，每个含 `pose.position` (xyz, 米) + `pose.orientation` (quaternion) + `confidence`

来源: `wuji-sdk/examples/python/wuji_glove/0.subscribe_callback.py:42-44`，`wuji-description/glove/body/urdf/right.urdf` (TX/RX coil links)

```
EmfPose 结构:
  pose.position: [x, y, z]  # 米，相对 wrist 坐标系
  pose.orientation: Quaternion(x, y, z, w)  # 归一化四元数
  confidence: float  # EMF 位姿估计可靠性 [0, 1]
```

### 1.2 在线 IK (Inverse Kinematics) 求解 — ✅ 已实现

EMF 指尖位姿 → 21-DOF 手关节角度的逆运动学求解：

- **输入**: 5 个指尖 EMF 位姿 + 腕部 IMU 方向 + URDF 手模型
- **输出**: `HandJointAngles` — 5 个手指各 4 个关节角度 (radians) + per-finger confidence
- **DOF 分配**: 拇指 5 (CMC_rot/flex/abd + MCP + IP) + 食指~小指各 4 (MCP_flex/abd + PIP + DIP) = 21 DOF ✅
- **并行化**: v0.7.0 起 IK 求解跨 CPU 核并行，降低 `hand_joint_angles` 计算延迟
- **速率分频**: `emf_poses_rate_divider` 可设 N，EMF 源 ~120Hz，下游全部降至 120/N Hz（N=4→30Hz），IK 运行次数减少 N 倍

来源: `wuji-sdk/CHANGELOG.md` v0.6.0/v0.7.0/v2026.6.16，`wuji-sdk/examples/c/wuji_glove/1_emf_poses_rate_divider.c`

**关节角数据结构** (Python):
```python
HandJointAngles:
  header: FrameHeader  # seq, timestamp_us, frame_id
  fingers: list[FingerAngles]  # 5 个手指
    # 每个 FingerAngles:
    #   angles: list[float]  # 4 个关节角 (radians)
    #   confidence: float    # 该手指 IK 置信度
```

### 1.3 IK 校准 (per-user hand model) — ✅ 已实现

- **目的**: 为每个用户生成个性化的 URDF 手模型（`left_hand.urdf` / `right_hand.urdf`），使 IK 求解匹配物理手尺寸
- **流程**: 引导用户做出一系列手势（pinch, four-finger bend 等），采集 EMF 数据，数值求解关节参数，生成 URDF
- **状态机**: `waiting_movement` → `waiting_stable` → `collecting` → `done`（per-pose），最后数值求解 + publish
- **校准约束**: `constraints_ok` + `metrics` 数组（per-finger 误差，单位 m，含 hint）
- **存储**: per SDK user + per hand side，同用户同侧的手套共享同一模型
- **热重载**: 校准完成后自动 hot-reload 在线 IK

来源: `wuji-sdk/examples/python/wuji_glove/5.calibration.py`，`wuji-cli/skills/wuji-cli-calibrate/SKILL.md`

### 1.4 触觉校准 (per-glove contact model) — ✅ 已实现

- **目的**: 为每只手套训练个性化的接触检测模型 (`tactile_binary`)
- **流程**: 引导用户做出 4 种动作（`four_finger_L_thumb_in`, `claw_curl`, `abduction`, `finger_to_palm`），采集触觉数据
- **训练**: 神经网络训练（`epochs` 参数，默认 60），输出 `best_val_loss`
- **模型**: per-glove SN，训练完成后自动 install 并在运行时自动加载
- **验证**: 报告 `alive_taxels`, `dead_pct`, `verified_alive_taxels`
- **灵敏度**: `tactile_binary_sensitivity` 可调 (0.5–3.0)，>1 更灵敏

来源: `wuji-sdk/examples/c/wuji_glove/5_tactile_calibration.c`，`wuji-sdk/examples/python/wuji_glove/7.tactile_calibration.py`

### 1.5 IMU 融合 — ✅ 已实现

- **6 个 IMU 传感器**: palm + thumb + index + middle + ring + pinky
- **每个 IMU 输出**: 加速度计 (3-axis) + 陀螺仪 (3-axis) + 融合方向 (quaternion) + orientation_covariance
- **腕部方向**: IMU 驱动的 waist → wrist 实时方向变换（`tf` stream）
- **静态变换**: wrist → emf_tx, wrist → palm_imu_link（`tf_static` stream, 1Hz）
- **离线管线**: `WujiGlove.offline_pipeline()` 可用合成 EMF/IMU 帧离线重跑 IK + IMU 融合

来源: `wuji-sdk/CHANGELOG.md` v0.6.0，`wuji-sdk/examples/python/wuji_glove/3.offline_pipeline.py`

```python
ImuData:
  header: FrameHeader
  angular_velocity: Vector3F64  # rad/s
  linear_acceleration: Vector3F64  # m/s^2
  orientation: Quaternion  # 融合后的方向
  orientation_covariance: list[float]  # 3x3 行优先
```

### 1.6 触觉残余信号 (tactile_residual) — ✅ 已实现

- **定义**: 校准基线去除后的连续有符号残差信号
  - `> 0`: 比校准基线更用力
  - `≈ 0`: 无接触
  - `< 0`: 比基线更轻
- **与 tactile_binary 的关系**: residual 是 binary 的底层连续信号；binary 用内置阈值二值化，residual 让用户自设阈值
- **掩码值**: `-1.0` 表示无效/掩码 taxel

来源: `wuji-sdk/examples/c/wuji_glove/7_tactile_residual_view.c`，`wuji-sdk/CHANGELOG.md` v2026.6.15

### 1.7 重定向 (Retargeting) — ✅ 已实现

- **输入**: 21 个 MediaPipe 格式手部关键点 (21×3, 米)
- **输出**: 20 关节命令 (radians, firmware 关节顺序)
- **接口**: `RetargetSession.for_hand(HandModel, side=Handedness)` → `session.step(keypoints)` → `(20,) qpos`
- **特性**: warm-start + smoothing 状态跨帧保持；`session.reset()` 用于追踪中断后重置
- **纯软件**: 不需要硬件，可作为任何 (21,3) 关键点源的下游
- **支持模型**: `HandModel.WujiHand` (一代) + `HandModel.WujiHand2`

来源: `wuji-sdk/examples/python/retargeting/0.retarget_session.py`，`wuji-sdk/examples/c/retargeting/0_retarget_session.c`

### 1.8 触觉点云 (tactile_point_cloud) — ✅ 已实现

- **v0.7.0 改进**: 用 mesh-based skinning deformation 替换几何近似，提升点云精度
- **输出**: `PointCloud` (3D 触觉可视化)

来源: `wuji-sdk/CHANGELOG.md` v0.7.0

---

## 2. 数据结构/通信协议

### 2.1 通信架构

```
                          ┌─────────────────────────────────────────────────────┐
                          │                   Host (PC / Jetson)                │
                          │                                                     │
                          │  ┌──────────┐    ┌──────────┐    ┌──────────────┐   │
                          │  │ Wuji CLI │    │ Wuji SDK │    │ Wuji Studio  │   │
                          │  │ (stateless)│   │ (Python) │    │ (GUI)        │   │
                          │  └────┬─────┘    └────┬─────┘    └──────┬───────┘   │
                          │       │               │                  │          │
                          │       └───────┬───────┴──────────────────┘          │
                          │               │                                     │
                          │      ┌────────▼────────┐                            │
                          │      │  Zenoh Bridge   │  ← 多客户端共享 (可选)      │
                          │      │  (enable_bridge)│                            │
                          │      └────────┬────────┘                            │
                          └───────────────┼─────────────────────────────────────┘
                                          │
                    ┌─────────────────────┼─────────────────────┐
                    │ UDP (局域网)         │ USB                  │
                    │ 192.168.x.x:50001   │ /dev/ttyACMx        │
                    └─────────────────────┼─────────────────────┘
                                          │
                          ┌───────────────▼───────────────┐
                          │       Wuji Glove / Hand 2      │
                          │  (ESP32? MCU + EMF + 触觉)     │
                          └───────────────────────────────┘
```

**关键通信特征**:
- **设备发现**: UDP 广播扫描 + USB 枚举（`wuji_scan()` / `manager.scan()` / `wuji devices`）
- **传输**: UDP (局域网, `ip:port`) 或 USB (`/dev/ttyACMx`)
- **单占用**: 设备固件同一时间只允许一个 direct session
- **多客户端**: SDK 可通过 Zenoh bridge 共享设备（`ConnectOptions(enable_bridge=True)` 默认）；bridge 模式下写入被拒
- **无连接状态**: CLI 每条命令独立 connect → operate → disconnect
- **SN 格式**: `WG1KXXXXXXXXXXX`（Glove SN 前缀 `WG1K`）

来源: `wuji-cli/skills/wuji-cli-base/SKILL.md`，`wuji-sdk/examples/python/wuji_glove/0.subscribe_callback.py:103-107`

### 2.2 SDK 数据流 (Topics)

| Topic | 类型 | 频率 | 内容 | 设备 |
|-------|------|------|------|------|
| `tactile` | TactileFrame | ~100Hz? | 744 值 (24×31) 校准触觉矩阵 | Glove |
| `tactile_zones` | TactileZones | — | 分区触觉 (palm, thumb, ...) | Glove |
| `tactile_binary` | TactileBinary | 同 tactile | 744 值二值接触 (1.0/0.0/-1.0) | Glove |
| `tactile_residual` | TactileResidual | 同 tactile | 744 值有符号残差 | Glove |
| `emf_poses` | EmfPoseArray | ~120Hz | 5 指尖 EMF 位姿 | Glove |
| `hand_joint_angles` | HandJointAngles | EMF/N | 21-DOF IK 关节角 | Glove |
| `hand_skeleton` | HandSkeleton | EMF/N | 21 MediaPipe landmarks | Glove |
| `tip_poses` | FingertipPoses | EMF/N | 指尖位姿 | Glove |
| `tactile_point_cloud` | PointCloud | EMF/N | 3D 触觉点云 | Glove |
| `imu_data/palm` | ImuData | — | 掌心 IMU (加速度+陀螺+方向) | Glove |
| `imu_data/thumb` ... `pinky` | ImuData | — | 指节 IMU (共 6 个) | Glove |
| `tf` | TransformArray | 实时 | 动态坐标变换 (waist→wrist) | 全局 |
| `tf_static` | TransformArray | 1Hz | 静态坐标变换 (wrist→emf_tx 等) | 全局 |
| `joint_states` | JointStateFrame | ~1kHz | 20 关节 position/velocity/effort | Hand 2 |
| `joint_diagnostics` | JointDiagnosticsFrame | — | 每关节 status_word/current/vbus/temp/error | Hand 2 |
| `fingertip/<finger>/data` | FingertipSensorData | 100Hz | 指尖力传感点阵 | Hand 2 |

来源: `wuji-sdk/examples/c/wuji_glove/0_subscribe_callback.c`，`wuji-cli/skills/wuji-cli-base/SKILL.md`，`wuji-sdk/examples/python/wuji_hand_2/3.fingertip_typed.py`

### 2.3 触觉数据格式 — ✅ 已实现

**重大变更 (v2026.7.14)**: 触觉矩阵从 768 (24×32) 缩减为 **744 (24×31)**，因设备端丢弃了 1 列死传感列。

```
触觉矩阵: 24 行 × 31 列 = 744 taxels
  布局: row-major, data[row * 31 + col]
  tactile / tactile_binary / tactile_residual 共享同一形状

值约定:
  tactile:        float, 校准后压力值
  tactile_binary: 1.0 = 接触, 0.0 = 无接触, -1.0 = 无效/掩码
  tactile_residual: 有符号 float, >0 硬于基线, ≈0 无, <0 轻于基线, -1.0 = 掩码
```

来源: `wuji-sdk/CHANGELOG.md` v2026.7.14，`wuji-sdk/examples/c/wuji_glove/6_tactile_contact_view.c` (`#define TACTILE_ROWS 24`, `#define TACTILE_COLS 31`)

**TactileFrame 结构** (C):
```c
WujiTactileFrame:
  WujiFrameHeader header;   // seq, timestamp_us, frame_id
  const float *data;        // 744 个 float 值
  size_t data_len;          // 744

WujiTactileZones:
  WujiFrameHeader header;
  const float *palm;  size_t palm_len;
  const float *thumb; size_t thumb_len;
  // ... 各分区独立数组
```

### 2.4 四元数格式 — ✅ 已实现

- **约定**: `Quaternion(x, y, z, w)` — w-last，与 EgoGlove Hand Token v2 的 w-first 不同
- **归一化**: 单位四元数
- **坐标系**: 待人工核对（URDF 隐含右手系，wrist 为根 link）

来源: `wuji-sdk/examples/python/wuji_glove/3.offline_pipeline.py` (`Quaternion(0.0, 0.0, 0.0, 1.0)` = identity)

> ⚠️ **EgoGlove 对比注意**: Wuji 用 w-last `(x,y,z,w)`，EgoGlove Hand Token v2 用 w-first `(w,x,y,z)`。互操作时需转换。

### 2.5 Wuji Hand 2 指尖触觉格式 — ✅ 已实现

指尖力传感采用**自描述数据帧**：格式由 `fingertip_info` 的 `format` JSON 字段动态定义，SDK 不硬编码布局。

```json
{
  "v": 1,
  "encoding": "point_array",
  "point_count": 34,  // thumb=40, 其他手指=34
  "point_stride": <bytes>,
  "aggregate_stride": <bytes>,
  "point_fields": [
    {"name": "fx", "type": "f32", "offset": 0, "scale": 1.0},
    {"name": "fy", "type": "f32", "offset": 4, "scale": 1.0},
    {"name": "fz", "type": "f32", "offset": 8, "scale": 1.0}
    // ... 更多 per-point 字段
  ],
  "aggregate_fields": [
    {"name": "temperature", "type": "f32", "offset": 0, "scale": 1.0},
    {"name": "fx", "type": "f32", ...},
    {"name": "fy", ...},
    {"name": "fz", ...}
  ]
}
```

- **每点**: 3D 力向量 `(fx, fy, fz)` (牛顿)
- **聚合**: 温度 + 总力向量
- **频率**: 100Hz
- **字段类型映射**: `i8/u8/i16/u16/i32/u32/f32` → struct format chars

来源: `wuji-sdk/examples/python/wuji_hand_2/3.fingertip_typed.py`

### 2.6 关节命令格式 (Hand 2) — ✅ 已实现

```python
# MIT 阻抗控制: 每关节 {position, velocity, effort}
JointCommand:
  position: float   # radians, 目标位置
  velocity: float   # rad/s, 前馈速度
  effort: float     # 力矩前馈

# 发送: 一次 20 关节帧
publisher.send([JointCommand(pos, vel, eff), ...])  # 20 个
```

- **频率**: 支持 200Hz 发布率
- **MIT 模式**: 固件默认，无需设 control_mode
- **参数**: `effort_limit` (Amps), `mit_params` (kp, kd) — 标量广播或 20 元素逐关节

来源: `wuji-sdk/examples/python/wuji_hand_2/2.publish.py`

### 2.7 关节状态格式 (Hand 2) — ✅ 已实现

```c
WujiJointStateFrame:
  WujiFrameHeader header;     // frame_id = "l_wrist"/"r_wrist"
  uint32_t num_joints;        // 在线关节数 (≤20)
  WujiJointStateEntry joints[];  // 变长，仅在线关节

WujiJointStateEntry:
  uint32_t nid;       // firmware 关节 ID
  float position;     // radians
  float velocity;     // rad/s
  float effort;       // 力矩代理
```

- **频率**: ~1kHz 广播
- **变长**: 只含固件本轮报告的在线关节，用 `nid` 识别
- **在线位图**: `WUJI_JOINT_ONLINE(online_mask, i)` 宏检查关节 i 是否在线

来源: `wuji-sdk/examples/c/wuji_hand_2/0_subscribe_joint_states.c`

### 2.8 MCAP 录制格式 — ✅ 已实现

- **格式**: MCAP (Foxglove 兼容)
- **压缩**: LZ4 / Zstd
- **多通道**: `TopicRecorder` 注册多个 `.subscribe()` 流
- **质量控制**: 帧丢率、抖动、跨通道同步偏移实时监控
- **片段切换**: 复用 session 配置，切换输出文件
- **暂停/恢复**: `RecordingHandle.pause()` / `resume()` / `stop()`

来源: `wuji-sdk/examples/python/wuji_glove/2.recording.py`，`wuji-sdk/CHANGELOG.md` v0.7.0

### 2.9 帧头 (FrameHeader) — ✅ 已实现

```c
WujiFrameHeader:
  uint32_t seq;           // 帧序号
  uint64_t timestamp_us;  // 微秒时间戳
  const char *frame_id;   // 坐标系名 (e.g. "r_hand_emf_tx", "l_wrist")
```

所有数据帧共享此通用头。

---

## 3. 硬件拓扑

### 3.1 Wuji Glove 传感器布局

```
┌─────────────────────────────────────────────────────────────────┐
│                      Wuji Glove (单手)                           │
│                                                                 │
│  ┌─────────┐  EMF TX 线圈 (腕部发射器)                           │
│  │  Wrist  │─── emf_tx_base_fixed ──→ emf_tx_base (TX coil)     │
│  │  IMU    │  (掌心 IMU: accel + gyro + fused quat)             │
│  └────┬────┘                                                     │
│       │                                                          │
│  ┌────▼──────────────────────────────────────────────────┐      │
│  │  5 × 手指 (thumb, index, middle, ring, pinky)         │      │
│  │                                                       │      │
│  │  每指:                                                │      │
│  │  ├── EMF RX 线圈 (指尖接收器)  ──→ <finger>_rx_coil   │      │
│  │  ├── 指节 IMU (6-axis, fused quat)                    │      │
│  │  ├── 触觉传感 (fingerpad_cp + fingertip_cp)           │      │
│  │  └── 关节: MCP_flex, MCP_abd, PIP, DIP (4 DOF)        │      │
│  │      (拇指: CMC_rot, CMC_flex, CMC_abd, MCP, IP = 5)  │      │
│  └───────────────────────────────────────────────────────┘      │
│                                                                 │
│  触觉矩阵: 24 × 31 = 744 taxels (全手分布)                       │
│  IMU 总数: 6 (palm + 5 fingers)                                 │
│  EMF: 1 TX + 5 RX = 6 线圈                                      │
│  URDF 关节: 21 revolute (5 thumb + 4×4 others)                 │
│                                                                 │
│  通信: UDP (局域网) 或 USB                                      │
│  SN 格式: WG1KXXXXXXXXXXX                                       │
└─────────────────────────────────────────────────────────────────┘
```

来源: `wuji-description/glove/body/urdf/right.urdf` (21 revolute joints + TX/RX coils + IMU links + tactile CP links)

**Glove URDF 关节拓扑** (21 revolute):

| 手指 | 关节名 | DOF |
|------|--------|-----|
| Thumb | cmc_rot, cmc_flex, cmc_abd, mcp, ip | 5 |
| Index | mcp_flex, mcp_abd, pip, dip | 4 |
| Middle | mcp_flex, mcp_abd, pip, dip | 4 |
| Ring | mcp_flex, mcp_abd, pip, dip | 4 |
| Pinky | mcp_flex, mcp_abd, pip, dip | 4 |
| **Total** | | **21** |

来源: `wuji-description/glove/body/urdf/right.urdf` — 21 revolute joints confirmed

**Glove URDF links** (40+):
- `wrist` (根) → `emf_tx_base` (TX 线圈)
- 每指: `metacarpal` → `proximal_rot` (thumb only) → `proximal_abd` → `proximal` → `middle` → `distal` → `tip`
- 每指: `rx_coil` (RX 线圈) + `fingerpad_cp` + `fingertip_cp` (触觉接触点)

> **注意**: Glove URDF 关节限位全部为 ±π (±3.1415926536)，effort=1.0, velocity=1.0 — 这是 IK 模型的运动学限位，非物理关节限位。

### 3.2 Wuji Hand (一代) 灵巧手拓扑

```
┌───────────────────────────────────────────────┐
│              Wuji Hand (一代)                  │
│                                               │
│  5 × 手指 (finger1=thumb ... finger5=pinky)   │
│  每指 4 关节: joint1, joint2, joint3, joint4   │
│  Total: 20 revolute joints                    │
│                                               │
│  通信: USB                                     │
│  控制: 低通实时控制器 (LowPass)                 │
│  SDK: Linux x86_64/aarch64 only (无 Android)   │
└───────────────────────────────────────────────┘
```

**Hand 1 URDF 关节** (20 revolute, `right_finger{1-5}_joint{1-4}`):

| 手指 | 关节 | 限位 (rad) | 力矩 (Nm) |
|------|------|-----------|----------|
| Finger1 (thumb) | joint1 | 0.0475 ~ 1.6033 | ±0.4452 |
| | joint2 | -0.1387 ~ 0.9324 | ±0.4259 |
| | joint3 | -0.4642 ~ 1.5623 | ±0.1888 |
| | joint4 | -0.4699 ~ 1.5568 | ±0.1468 |
| Finger2-5 | joint1 | ~-0.16 ~ ~1.56 | ±0.62~0.65 |
| | joint2 | -0.37 ~ 0.37 | ±0.18~0.18 |
| | joint3 | ~-0.48 ~ ~1.55 | ±0.19~0.24 |
| | joint4 | ~-0.48 ~ ~1.57 | ±0.15~0.22 |

来源: `wuji-description/hand/body/urdf/right.urdf`，`wuji-description/hand/body/mjcf/right.xml`，`wuji-description/hand/body-with-soft/params.csv`

### 3.3 Wuji Hand 2 (Beta 2) 灵巧手拓扑

```
┌───────────────────────────────────────────────────────┐
│              Wuji Hand 2 (Beta 2)                      │
│                                                       │
│  5 × 手指 (anatomical naming)                         │
│  每指 4 revolute + 2 fixed = 20 actuated DOF          │
│  + 每指尖 1 × 触觉传感垫 (tip_sensor_frame)           │
│                                                       │
│  关节命名 (右侧):                                     │
│    r_thumb:    cmc_flex, cmc_abd, mcp, ip             │
│    r_index:    mcp_flex, mcp_abd, pip, dip            │
│    r_middle:   mcp_flex, mcp_abd, pip, dip            │
│    r_ring:     mcp_flex, mcp_abd, pip, dip            │
│    r_pinky:    mcp_flex, mcp_abd, pip, dip            │
│                                                       │
│  通信: Ethernet (EtherCAT? 待人工核对)                 │
│  控制: MIT 阻抗 (kp, kd, effort_limit)                │
│  诊断: 每关节 status_word, current, vbus, temp, error │
│  总线健康: response rate, timeout count, stream loss  │
│  指尖触觉: thumb=40 points, others=34 points, 100Hz  │
│                                                       │
│  变体: Beta 1 (无指尖触觉) / Beta 2 (有指尖触觉)      │
│  安装: _with_mount 变体含 palm 安装法兰                │
│  适配器: Unitree G1 / RL 开源底座 / 抗冲击附件         │
└───────────────────────────────────────────────────────┘
```

**Hand 2 URDF 关节** (20 revolute, anatomical names):

| 手指 | 关节 | 类型 |
|------|------|------|
| Thumb | r_thumb_cmc_flex, r_thumb_cmc_abd, r_thumb_mcp, r_thumb_ip | revolute |
| Index | r_index_finger_mcp_flex, r_index_finger_mcp_abd, r_index_finger_pip, r_index_finger_dip | revolute |
| Middle | r_middle_finger_mcp_flex, r_middle_finger_mcp_abd, r_middle_finger_pip, r_middle_finger_dip | revolute |
| Ring | r_ring_finger_mcp_flex, r_ring_finger_mcp_abd, r_ring_finger_pip, r_ring_finger_dip | revolute |
| Pinky | r_pinky_mcp_flex, r_pinky_mcp_abd, r_pinky_pip, r_pinky_dip | revolute |
| **Total** | | **20** |

来源: `wuji-description/hand2/hand2_beta2/body/urdf/right.urdf`

**Hand 2 额外 links** (每指):
- `<finger>_tip` (fixed) → `<finger>_tip_sensor_frame` (fixed, 触觉传感垫)
- `_with_mount` 变体额外含 `{l,r}_mount.STL`

### 3.4 关节参数表 (Hand 1 with-soft, params.csv) — ✅ 已实现

完整 20 关节的 kp/kv/armature/damping/effort_limit/ctrl_range 参数表已提供（MuJoCo 模型用），是商业级标定数据。摘录:

| idx | name | kp | kv | effort_limit (Nm) | ctrl_range (rad) |
|-----|------|-----|-----|---------|---------|
| 0 | finger1_joint1 | 0.408 | 0.021 | 0.4452 | 0.037~1.613 |
| 1 | finger1_joint2 | 0.686 | 0.031 | 0.4259 | -0.158~0.931 |
| 4 | finger2_joint1 | 0.374 | 0.019 | 0.6188 | -0.161~1.559 |
| 5 | finger2_joint2 | 0.456 | 0.020 | 0.1822 | -0.404~0.305 |
| ... | ... | ... | ... | ... | ... |

来源: `wuji-description/hand/body-with-soft/params.csv`

---

## 4. 关键 API 签名

### 4.1 Python SDK — 设备管理

```python
# 设备发现
manager = SdkManager.instance()
devices: list[DiscoveredDevice] = manager.scan()
# DiscoveredDevice: .sn (str), .device_type (DeviceType), .address (str)

# 连接 (多客户端默认)
glove: WujiGlove = manager.connect(
    sn: str,
    device_name: str,
    options: ConnectOptions = ConnectOptions()  # enable_bridge=True 默认
)

# 设备类型枚举
DeviceType.WujiGlove | DeviceType.WujiHand | DeviceType.WujiHand2

# 连接选项
ConnectOptions(enable_bridge: bool = True)  # False = 独占模式
ConnectOptions(auto_time_sync_interval_ms: int | None = 30000)  # None=禁用
```

来源: `wuji-sdk/examples/python/wuji_glove/0.subscribe_callback.py`，`wuji-sdk/CHANGELOG.md` v2026.5.26

### 4.2 Python SDK — WujiGlove 数据流

```python
# 订阅 (回调模式)
sub = glove.tactile().subscribe_with_callback(callback: Callable[[TactileFrame], None])
sub = glove.tactile_zones().subscribe_with_callback(callback)
sub = glove.emf_poses().subscribe_with_callback(callback)
sub = glove.hand_joint_angles().subscribe_with_callback(callback)
sub = glove.hand_skeleton().subscribe_with_callback(callback)
sub = glove.tactile_point_cloud().subscribe_with_callback(callback)
sub = glove.tactile_binary().subscribe_with_callback(callback)
sub = glove.tactile_residual().subscribe_with_callback(callback)

# 订阅 (异步模式)
sub = glove.tactile().subscribe()
frame: TactileFrame = await sub.recv_async()

# IMU (6 个)
sub = glove.imu_palm().subscribe()
sub = glove.imu_thumb().subscribe()
sub = glove.imu_index().subscribe()
sub = glove.imu_middle().subscribe()
sub = glove.imu_ring().subscribe()
sub = glove.imu_pinky().subscribe()

# 坐标变换 (全局, 非设备级)
manager.subscribe("tf")         # 动态
manager.subscribe("tf_static")  # 静态

# 参数读写
glove.hand_model_path().set(str)  # 自定义 IK URDF
glove.hand_model_path().get() -> str
glove.emf_poses_rate_divider().set(int)  # EMF 速率分频 N
glove.emf_poses_rate_divider().get() -> int
glove.save_params()  # 持久化到 flash
glove.sync_time()  # 手动时间同步

# 校准
result = await glove.calibrate(
    skip_constraints: bool,
    timeout_s: float,
    on_feedback: Callable[[dict], None]
) -> dict
result = glove.calibrate_blocking(...)  # 阻塞版
result = glove.calibrate_tactile_blocking(
    seconds_per_pose: float,
    epochs: int,
    install: bool,
    sensitivity: float | None,
    timeout_s: float,
    on_feedback: Callable,
    on_pose_prompt: Callable
) -> dict

# 离线管线
pipeline = WujiGlove.offline_pipeline(sn: str, hand_side: str, urdf_path: str | None)
pipeline.hand_joint_angles().compute(emf_frame: EmfPoseArray) -> HandJointAngles
pipeline.tip_poses().compute(emf_frame) -> FingertipPoses
pipeline.hand_skeleton().compute(emf_frame) -> HandSkeleton
pipeline.imu_data_palm().compute(imu: ImuData) -> ImuData  # IMU 融合

# 设备属性
glove.serial_number -> str
glove.device_name -> str
glove.version().get() -> str  # 固件版本
glove.hand_side().get() -> str  # "left"/"right"
```

来源: `wuji-sdk/examples/python/wuji_glove/*.py`，`wuji-sdk/CHANGELOG.md`

### 4.3 Python SDK — WujiHand2 控制

```python
hand: WujiHand2 = manager.connect(sn=..., device_name=...)

# 状态
hand.online_joints_count().get() -> int  # 在线关节数 (0-20)
hand.handedness  # 属性

# 控制 (MIT 阻抗)
hand.effort_limit().set(float | list[float])  # 标量广播或 20 逐关节
hand.mit_params().set(tuple[float, float] | list[tuple])  # (kp, kd)
hand.enable()
hand.disable()
hand.clear_fault()  # 清除所有关节故障 (v2026.7.14 起无参数)
hand.reboot()

# 关节命令发布
publisher = hand.joint_command().publish()
publisher.send([JointCommand(position, velocity, effort), ...])  # 20 个/帧

# 数据流
hand.joint_states().subscribe() -> Subscription[JointStateFrame]
hand.joint_diagnostics().subscribe() -> Subscription[JointDiagnosticsFrame]

# 指尖触觉
hand.get_fingertip_info(finger_idx: int) -> FingertipSensorInfo  # 含 format JSON
hand.fingertip_thumb_data().subscribe() -> Subscription[FingertipSensorData]
# ... index/middle/ring/pinky
```

来源: `wuji-sdk/examples/python/wuji_hand_2/*.py`，`wuji-sdk/CHANGELOG.md` v2026.7.14

### 4.4 Python SDK — Retargeting

```python
from wuji_sdk import HandModel, Handedness, RetargetSession

# 创建 session (选模型 + 选手)
session = RetargetSession.for_hand(
    HandModel.WujiHand2,  # 或 HandModel.WujiHand
    side=Handedness.Right  # 或 Handedness.Left
)

# 每帧调用
qpos: np.ndarray = session.step(keypoints: np.ndarray)  # (21,3) float32 → (20,) float32
# qpos 为 firmware 关节顺序, radians

# 追踪中断后重置
session.reset()
```

来源: `wuji-sdk/examples/python/retargeting/0.retarget_session.py`

### 4.5 C SDK — 核心函数

```c
// 初始化
WujiStatus wuji_init(const WujiInitOptions *opts);
const char *wuji_version(void);
const char *wuji_last_error(void);
void wuji_shutdown(void);

// 设备发现
WujiStatus wuji_scan(WujiDiscovered **list, size_t *n);
void wuji_discovered_free(WujiDiscovered *list, size_t n);
// WujiDiscovered: .serial_number, .model, .address, .device_id

// 连接
WujiStatus wuji_connect(
    const WujiConnectTarget *target,  // .kind=SN/ADDRESS, .value
    const char *alias,
    const WujiConnectOptions *opts,   // .timeout_ms, .retry_count, .enable_bridge
    WujiDevice **dev
);
WujiConnectOptions wuji_connect_options_default(void);
void wuji_dev_disconnect(WujiDevice *dev);
void wuji_dev_release(WujiDevice *dev);

// 订阅 (typed callbacks, 每流一线程)
WujiStatus wuji_glove_subscribe_tactile(WujiDevice*, WujiTactileSubCallback, void*, WujiSub**);
WujiStatus wuji_glove_subscribe_tactile_binary(WujiDevice*, WujiTactileBinarySubCallback, void*, WujiSub**);
WujiStatus wuji_glove_subscribe_tactile_residual(WujiDevice*, WujiTactileResidualSubCallback, void*, WujiSub**);
WujiStatus wuji_glove_subscribe_emf_poses(WujiDevice*, WujiEmfPosesSubCallback, void*, WujiSub**);
WujiStatus wuji_glove_subscribe_hand_joint_angles(WujiDevice*, WujiJointAnglesSubCallback, void*, WujiSub**);
WujiStatus wuji_glove_subscribe_hand_skeleton(WujiDevice*, WujiSkeletonSubCallback, void*, WujiSub**);
WujiStatus wuji_glove_subscribe_tactile_point_cloud(WujiDevice*, WujiPointCloudSubCallback, void*, WujiSub**);

// 全局变换
WujiStatus wuji_subscribe_tf(WujiTransformSubCallback, void*, WujiSub**);
WujiStatus wuji_subscribe_tf_static(WujiTransformSubCallback, void*, WujiSub**);

// 参数
WujiStatus wuji_glove_set_emf_poses_rate_divider(WujiDevice*, uint32_t n);
WujiStatus wuji_glove_set_tactile_binary_sensitivity(WujiDevice*, double value);

// 校准
WujiStatus wuji_glove_calibrate_tactile_blocking(
    WujiDevice*, const WujiTactileCalibrationOptions*,
    const WujiTactileCalibrationCallbacks*, WujiTactileCalibrationSummary*);
WujiStatus wuji_glove_calibrate_blocking(WujiDevice*, ...);

// Hand 2
WujiStatus wuji_hand_2_online_joints_count(WujiDevice*, uint8_t*);
WujiStatus wuji_hand_2_get_handedness(WujiDevice*, WujiHandedness*);
WujiStatus wuji_hand_2_get_all_effort_limit(WujiDevice*, float[20], uint32_t* online_mask);
WujiStatus wuji_hand_2_set_all_effort_limit(WujiDevice*, float);
WujiStatus wuji_hand_2_set_all_mit_params(WujiDevice*, const float kp[20], const float kd[20]);
WujiStatus wuji_hand_2_enable(WujiDevice*, const uint8_t* mask);  // NULL=全手
WujiStatus wuji_hand_2_disable(WujiDevice*, const uint8_t* mask);
WujiStatus wuji_hand_2_joint_command_publish(WujiDevice*, WujiJointCommandPublisher**);
WujiStatus wuji_joint_command_publisher_send(WujiJointCommandPublisher*, const WujiJointCommand[20]);
WujiStatus wuji_hand_2_joint_label(uint8_t idx, char* buf, size_t buf_len, size_t*);
// 常量: WUJI_HAND_2_JOINT_COUNT = 20
// 宏:   WUJI_JOINT_ONLINE(mask, i)

// Retargeting
WujiStatus wuji_retarget_session_create(WujiHandModel, WujiHandedness, WujiRetargetSession**);
WujiStatus wuji_retarget_session_step(WujiRetargetSession*, const float kp[63], float qpos[20]);
WujiStatus wuji_retarget_session_reset(WujiRetargetSession*);
void wuji_retarget_session_free(WujiRetargetSession*);

// 帧类型枚举
WujiFrameKind: WUJI_FRAME_KIND_OK | LAG | END | ERROR
WujiStatus: WUJI_STATUS_OK | ERR_UNSUPPORTED | ERR_BUFFER_TOO_SMALL | ...
```

来源: `wuji-sdk/examples/c/wuji_glove/*.c`，`wuji-sdk/examples/c/wuji_hand_2/*.c`，`wuji-sdk/examples/c/retargeting/*.c`

### 4.6 CLI 命令

```bash
# 设备管理
wuji devices [--json]                          # 扫描列出设备
wuji ping [--sn <SN>] [--json]                 # 握手探测

# 参数读写
wuji resources --sn <SN> [--json]              # 列出可读参数和可订阅 topic
wuji get <param> --sn <SN> [--json]            # 读参数
wuji set <param> <value> --sn <SN>             # 写参数 (JSON 值, 0x=hex)

# 数据订阅
wuji sub <topic> --sn <SN> [--count N] [--jsonl]  # 实时数据
wuji sub tactile --count 500 --jsonl > data.jsonl  # 录制

# 校准
wuji calib ik [--sn <SN>] [--jsonl]            # IK 手模型校准
wuji calib tactile --sn <SN> [--non-interactive]  # 触觉校准

# 诊断
wuji doctor [--sn <SN>] [-v] [--json]          # 健康检查

# 固件升级
wuji upgrade --check                            # 检查更新
wuji upgrade --all                              # 全部升级
wuji upgrade --sn <SN> --to <version>           # 指定版本
wuji upgrade --file <path>                      # 本地包

# 用户管理
wuji user list                                  # 列出校准 profile
wuji user create <name> [-d <desc>] [--switch]  # 创建
wuji user switch <name>                         # 切换
wuji user export <path>                         # 导出校准
wuji user import <path> [--as <name>]           # 导入

# 日志
wuji logs path                                  # 日志目录
wuji logs list [--days N] [--source sdk,studio] # 列出
wuji logs export [-o <path>] [--days N]         # 导出 bundle

# 自更新
wuji update [--check]                           # CLI 自更新

# Shell 补全
wuji completions <bash|zsh|fish|powershell|elvish>

# 设备选择器 (互斥)
--sn <SERIAL>          # 序列号 (推荐)
--address <ip:port>    # 地址
--handedness <left|right>  # 手性
```

来源: `wuji-cli/skills/wuji-cli-base/SKILL.md`，`wuji-cli/README.md`

---

## 5. 工程陷阱

### 5.1 触觉矩阵尺寸变更 — ⚠️ Breaking Change

v2026.7.14 将触觉矩阵从 768 (24×32) 缩减为 **744 (24×31)**。下游代码若硬编码 768 或 24×32 会越界。`tactile`, `tactile_binary`, `tactile_residual`, `tactile_zones` 全部受影响。应始终用 `frame.data_len` 而非硬编码尺寸。

来源: `wuji-sdk/CHANGELOG.md` v2026.7.14

### 5.2 四元数顺序: w-last vs w-first

Wuji SDK 使用 `Quaternion(x, y, z, w)` (w-last)。EgoGlove Hand Token v2 使用 `(w, x, y, z)` (w-first)。互操作时**必须转换**，否则方向完全错误。

来源: `wuji-sdk/examples/python/wuji_glove/3.offline_pipeline.py` (`Quaternion(0.0, 0.0, 0.0, 1.0)` identity)

### 5.3 回调线程模型 — ⚠️ 并发陷阱

C SDK 的 typed 回调在**专用 worker 线程**上运行（每订阅一个线程），不是调用线程。同一订阅内的回调串行有序；不同订阅的回调可能并发。跨回调共享状态需自行同步。回调中的 frame 指针仅在回调期间有效（堆数据回调返回后立即释放），不可存储。

来源: `wuji-sdk/examples/c/wuji_glove/0_subscribe_callback.c` (THREADING note)

### 5.4 IK 校准要求命名用户

默认 SDK 用户无法进行 IK 校准。必须先 `wuji user create <name> --switch` 创建命名 profile，否则校准在收集任何 pose 前就以 exit 5 拒绝。触觉校准不要求命名用户（per-SN 而非 per-user）。

来源: `wuji-cli/skills/wuji-cli-user/SKILL.md`，`wuji-cli/skills/wuji-cli-calibrate/SKILL.md`

### 5.5 单占用 + Zenoh 桥接

设备固件同一时间只允许一个 direct session。当 Wuji Studio 等占用设备时，CLI/SDK 自动降级为 Zenoh bridge 只读模式（写入被拒 `direct_only`）。需独占时设 `ConnectOptions(enable_bridge=False)`。

来源: `wuji-cli/skills/wuji-cli-base/SKILL.md`

### 5.6 触觉校准需 model-export 构建

触觉校准的**训练**步骤需要 SDK 的 model-export 构建变体。标准 pip 包可能不含训练能力（C SDK 中返回 `WUJI_STATUS_ERR_UNSUPPORTED`）。校准后模型自动加载，但首次需用支持训练的构建。

来源: `wuji-sdk/CHANGELOG.md` v2026.8.3，`wuji-sdk/examples/c/wuji_glove/5_tactile_calibration.c`

### 5.7 EMF 下游速率耦合

`emf_poses_rate_divider` 设 N 后，**所有** EMF 派生流 (`emf_poses`, `hand_joint_angles`, `tip_poses`, `hand_skeleton`, `tactile_point_cloud`) 同时降至 120/N Hz。IMU 流和原始触觉流不受影响。设大 N 可减 CPU 但降追踪帧率。

来源: `wuji-sdk/CHANGELOG.md` v2026.6.16

### 5.8 Hand 2 关节变长帧

`joint_states` 和 `joint_diagnostics` 帧是变长的，只含在线关节。用 `nid` 识别每关节，不要按索引假设固定位置。`WUJI_JOINT_ONLINE(online_mask, i)` 检查关节是否在线。

来源: `wuji-sdk/examples/c/wuji_hand_2/0_subscribe_joint_states.c`

### 5.9 teleop 读取关键点需排空队列

Glove 的 `hand_skeleton` 发布快于消费循环，`recv()` 返回最旧未读帧。teleop 循环中必须 drain 队列只保留最新帧，否则逐帧消费会落后。

来源: `wuji-sdk/examples/python/retargeting/1.teleop_real.py` (`read_keypoints` 函数)

### 5.10 网络参数需重启

`wuji set ip_address` / `wuji set data_port` 成功后，设备需重启才生效。`wuji devices` 不会立即反映变更。

来源: `wuji-cli/skills/wuji-cli-base/SKILL.md`

### 5.11 C SDK ABI 变更

v2026.8.3 的 `WujiJointDiagnosticsEntry` / `WujiJointDiagnosticsFrame` 结构体新增字段，C ABI 变更。旧应用必须重新编译头文件后才能链接新库。

来源: `wuji-sdk/CHANGELOG.md` v2026.8.3

### 5.12 Android 不支持 Wuji Hand

C SDK 的 Android tarball 中 Wuji Hand (一代) 函数存在但返回 `WUJI_STATUS_ERR_UNSUPPORTED`。仅 Linux x86_64/aarch64 gnu 支持灵巧手控制。Retargeting 同样仅 Linux。

来源: `wuji-sdk/examples/c/README.md`

### 5.13 MCP 支持现状

> **待人工核对**: 任务描述提到 "Wuji supports MCP integration"，但在三个 repo 的源码和文档中未发现任何 MCP (Model Context Protocol) 相关代码或文档。MCP 集成可能存在于：
> - 在线文档中未抓取的页面
> - Wuji Studio 软件（非开源）
> - 独立的 MCP server 仓库
> - CLI 的 agent skills 被描述为 "skills" 而非 MCP，但安装方式 (`npx skills add`) 暗示与 agent 生态有关
>
> **结论**: repo 层面无 MCP 证据。Wuji CLI 的 "agent skills" 是 SKILL.md 格式的技能包（类似 Claude/Codex skills），非 MCP server。MCP 支持待人工核对在线文档或联系 support@wuji.tech 确认。

---

## 6. 对比表: Wuji Glove vs EgoGlove Lite / Pro

| 维度 | Wuji Glove | EchoGlove Lite | EchoGlove Pro |
|------|-----------|---------------|--------------|
| **DOF (追踪)** | 21 (IK from EMF) ✅ | 21 (IK from flex+IMU) 🟡 | 21+ (多 IMU 路线图) 🔬 |
| **传感方式** | EMF (电磁定位) + 6 IMU ✅ | flex(5) + 单腕 IMU ✅ | flex/eSkin + 工业级 IMU 🟡 |
| **EMF 线圈** | 1 TX + 5 RX ✅ | ❌ 无 | ❌ 无 (非 EMF 路线) |
| **触觉** | 744 taxels (24×31) 全手 ✅ | ❌ 无 | 🔬 力接口 (路线图) |
| **触觉校准** | 神经网络 per-glove 模型 ✅ | ❌ | 🔬 |
| **指尖力传感** | Hand 2: 34-40 points/finger ✅ | ❌ | 🔬 |
| **成本档位** | 商业级 (待人工核对, 预估 ¥万级) | <¥500 BOM ✅ | 待定 🟡 |
| **通信协议** | UDP (局域网) / USB ✅ | BLE / WiFi 🟡 | USB-C/WiFi/BT + ROS2/Ethernet 🟡 |
| **SDK 语言** | Python (pip) + C (prebuilt) ✅ | Python (规划) 🟡 | Python + C 🟡 |
| **SDK 平台** | Linux x86_64/aarch64 + Android ✅ | ESP32 → Phone/PC 🟡 | Linux + ROS2 🟡 |
| **CLI 工具** | wuji CLI (全功能) ✅ | ❌ 无 | 🟡 规划 |
| **URDF/MJCF/USD** | 三格式 + STEP + ROS2 包 ✅ | 🟡 规划 | 🟡 规划 |
| **重定向** | RetargetSession (21kp→20 joint) ✅ | 🟡 DexRetargeting | 🟡 DexRetargeting |
| **MCAP 录制** | 多通道 LZ4/Zstd ✅ | 🟡 规划 | 🟡 规划 |
| **MCP 支持** | 待人工核对 (repo 无证据) | 🟡 规划 | 🟡 规划 |
| **灵巧手** | Wuji Hand / Hand 2 (20 DOF) ✅ | ❌ (非本体) | ❌ (非本体) |
| **遥操作闭环** | Glove→Retarget→Hand 全链 ✅ | 🟡 (Glove→Hand Token→Robot) | 🟡 |
| **校准体系** | IK per-user + tactile per-glove ✅ | 🟡 flex 校准 | 🟡 多级校准 |
| **多客户端** | Zenoh bridge 共享 ✅ | ❌ | 🟡 |
| **在线文档** | docs.wuji.tech 完整 ✅ | docs/V7/ 内部 🟡 | 同 Lite |
| **市场定位** | 商业手套+灵巧手一体化 (中国) | 开放手部运动基础设施 | 具身智能数据入口 |
| **开源** | MIT (SDK+CLI+description) ✅ | 开源核心 🟡 | 商业数据层 |
| **生态锚点** | 自有 SDK 闭环 | MANO/OpenXR/ROS2/FreeMoCap 枢纽 | 同 Lite |

### 关键差异分析

**1. 传感路线根本不同**
Wuji 走 **EMF 电磁定位**路线（指尖 RX 线圈感应腕部 TX 电磁场），EgoGlove 走 **flex 弯曲 + IMU**路线。EMF 的优势是不依赖弯曲传感器精度、直接解算指尖空间位姿；flex 的优势是低成本、简单可靠。EgoGlove D12 已明确"不启动 IMU 阵列竞争"，定位为开放基础设施而非硬件竞品。

**2. Wuji 是"手套+灵巧手"垂直整合**
Wuji 同时做 Glove（输入）和 Hand/Hand 2（输出），提供 glove→retarget→hand 全链遥操作。EgoGlove 不做机器人本体，定位为 Hand Token 中间表示层，通过 DexRetargeting/AnyTeleop 桥接第三方灵巧手。

**3. 触觉是 Wuji 的差异化壁垒**
744 taxels 全手触觉 + 神经网络校准 + residual 连续信号 + Hand 2 指尖 34-40 点力传感，是 EgoGlove Lite 完全没有的能力。EgoGlove Pro 的力接口仍在路线图 (🔬)。

**4. 关节表示: 21 vs 20**
Wuji Glove URDF 是 **21 revolute**（拇指 5 DOF + 四指 4 DOF），Wuji Hand/Hand 2 是 **20 revolute**（每指 4 DOF）。EgoGlove Hand Token v2 canonical 是 **20 旋转关节**（腕1 + 拇指3 + 四指×4），派生 21 MediaPipe。Wuji Glove 的 21 比 EgoGlove canonical 多 1 个拇指 DOF (CMC_rot)，互操作时需注意映射。

**5. 四元数顺序冲突**
Wuji: `(x, y, z, w)` w-last。EgoGlove: `(w, x, y, z)` w-first。互操作时必须转换。

**6. 开放性 vs 封闭性**
Wuji SDK/CLI/Description 全部 MIT 开源，但核心 SDK 是 **prebuilt 二进制**（C SDK）或 **pip 包**（Python），不是源码开放。EgoGlove 定位为开放基础设施，协议层开源。Wuji 的 agent skills (SKILL.md) 模式值得参考——用结构化技能包让 AI agent 操作设备。

---

## 7. EgoGlove 可借鉴要点

### 7.1 SDK API 设计模式
- **语义 API**: `glove.tactile().subscribe_with_callback(cb)` — 流名即方法名，类型安全
- **双订阅模式**: callback (后台线程) + async/await (`recv_async()`) 覆盖不同使用场景
- **变长自描述帧**: Hand 2 指尖触觉用 `format` JSON 动态定义布局，SDK 不硬编码 — 适合 EgoGlove 的 capability-flagged TLV 设计
- **速率分频**: 单参数控制高代价流的下游速率，简洁有效

### 7.2 CLI 工具设计
- **无状态**: 每命令独立 connect→operate→disconnect，无持久连接
- **--json/--jsonl**: 结构化输出供脚本和 AI agent 消费
- **设备选择器**: `--sn` / `--address` / `--handedness` 三选一
- **agent skills**: SKILL.md 格式技能包，让 AI agent 理解 CLI 语义

### 7.3 校准体系
- **per-user IK + per-glove tactile**: 双层校准数据隔离
- **profile 管理**: create/switch/export/import，可跨机器迁移
- **引导式流程**: pose-by-pose 引导 + 实时质量反馈 + 误差 metrics

### 7.4 描述包 (Description)
- **多格式**: URDF + MJCF + USD + STEP + ROS2 package，一站式
- **变体管理**: with-mount / with-soft / simplified 变体覆盖不同仿真需求
- **CAD 集成**: STEP + PDF 图纸 + BOM 支持机械集成

### 7.5 MCAP 录制
- 多通道 + LZ4/Zstd + 质量监控 + 片段切换 — 适合 EgoGlove 数据采集场景

---

## 8. 待人工核对事项

| 事项 | 原因 | 查证方式 |
|------|------|---------|
| MCP 支持 | repo 无证据，任务描述提及 | 联系 support@wuji.tech 或查 docs.wuji.tech 搜索 "MCP" |
| 触觉矩阵分区映射 | `tactile_zones` 的 palm/thumb 分区对应哪些 row/col | docs.wuji.tech 触觉数据参考页 |
| EMF 采样率精确值 | 代码注释 "~120Hz"，未在协议文档中确认 | docs.wuji.tech SDK 数据参考 |
| Hand 2 通信总线 | CHANGELOG 提 "Ethernet stream loss"，未明确 EtherCAT/TCP/UDP | docs.wuji.tech Hand 2 文档 |
| Wuji Glove 售价 | repo 无价格信息 | wuji.tech 官网或销售渠道 |
| MCU 型号 | repo 无固件源码，无法确认 ESP32 还是其他 | 拆解或 docs.wuji.tech 硬件页 |
| 触觉传感技术 | 744 taxels 的物理原理 (压阻/电容/磁觉) | docs.wuji.tech 硬件页 |
| IMU 型号 | 6 个 IMU 的具体芯片型号 | docs.wuji.tech 硬件页 |
| Hand 2 关节限位 | Beta 2 URDF 有关节限位但本次未提取全部 | 直接查看 URDF 文件 |

---

## 9. 文件索引

### wuji-sdk
| 文件 | 内容 |
|------|------|
| `README.md` | SDK 总览，Python/C 双语言 |
| `CHANGELOG.md` | 完整版本历史 (v0.5.0 → v2026.8.3) |
| `examples/python/README.md` | Python SDK 安装与快速上手 |
| `examples/python/wuji_glove/0.subscribe_callback.py` | 回调订阅全流 |
| `examples/python/wuji_glove/1.subscribe_async.py` | 异步订阅全流 |
| `examples/python/wuji_glove/2.recording.py` | MCAP 录制 |
| `examples/python/wuji_glove/3.offline_pipeline.py` | 离线 IK + IMU 融合 |
| `examples/python/wuji_glove/4.user.py` | 用户 profile 管理 |
| `examples/python/wuji_glove/5.calibration.py` | IK 校准 (terminal + API) |
| `examples/python/wuji_glove/7.tactile_calibration.py` | 触觉校准 |
| `examples/python/wuji_glove/8.tactile_contact_view.py` | tactile_binary 可视化 |
| `examples/python/wuji_glove/9.tactile_residual_view.py` | tactile_residual 可视化 |
| `examples/python/wuji_hand_2/0.subscribe_callback.py` | Hand 2 joint_states |
| `examples/python/wuji_hand_2/2.publish.py` | Hand 2 MIT 控制 |
| `examples/python/wuji_hand_2/3.fingertip_typed.py` | Hand 2 指尖触觉 |
| `examples/python/retargeting/0.retarget_session.py` | 重定向 (无硬件) |
| `examples/python/retargeting/1.teleop_real.py` | 实时遥操作全链 |
| `examples/c/README.md` | C SDK 下载与编译 |
| `examples/c/wuji_glove/0_subscribe_callback.c` | C 回调订阅 |
| `examples/c/wuji_glove/1_emf_poses_rate_divider.c` | EMF 速率分频 |
| `examples/c/wuji_glove/5_tactile_calibration.c` | C 触觉校准 |
| `examples/c/wuji_glove/6_tactile_contact_view.c` | C tactile_binary (24×31) |
| `examples/c/wuji_glove/7_tactile_residual_view.c` | C tactile_residual |
| `examples/c/wuji_hand_2/0_subscribe_joint_states.c` | C Hand 2 joint_states |
| `examples/c/wuji_hand_2/1_control_motion.c` | C Hand 2 MIT sweep |
| `examples/c/wuji_hand_2/2_hand_info.c` | C Hand 2 诊断 |
| `examples/c/retargeting/0_retarget_session.c` | C 重定向 |

### wuji-cli
| 文件 | 内容 |
|------|------|
| `README.md` | CLI 总览与快速上手 |
| `CHANGELOG.md` | 版本历史 (v2026.7.14 → v2026.8.3) |
| `skills/wuji-cli-base/SKILL.md` | 基础命令 (devices/ping/get/set/sub) |
| `skills/wuji-cli-user/SKILL.md` | 用户 profile 管理 |
| `skills/wuji-cli-doctor/SKILL.md` | 健康诊断 |
| `skills/wuji-cli-doctor/references/doctor.json` | 诊断 JSON 示例 |
| `skills/wuji-cli-calibrate/SKILL.md` | IK + 触觉校准引导 |
| `skills/wuji-cli-upgrade/SKILL.md` | 固件升级 |
| `skills/wuji-cli-logs/SKILL.md` | 日志导出 |
| `scripts/install-cli.sh` | CLI 安装脚本 |
| `scripts/install-skills.sh` | Skills 安装脚本 |

### wuji-description
| 文件 | 内容 |
|------|------|
| `README.md` | 包总览 (hand/hand2/glove/attachment) |
| `glove/body/urdf/right.urdf` | Glove URDF (21 revolute + EMF coils) |
| `hand/body/urdf/right.urdf` | Hand 1 URDF (20 revolute) |
| `hand/body/mjcf/right.xml` | Hand 1 MuJoCo (含 kp/kv/forcerange) |
| `hand/body-with-soft/params.csv` | Hand 1 全关节参数表 |
| `hand2/hand2_beta2/body/urdf/right.urdf` | Hand 2 Beta 2 URDF (20 revolute + tip sensor) |
| `hand2/hand2_beta1/body/package.xml` | ROS2 包 `wuji_hand2_description` |
| `hand2/hand2_beta2/body/package.xml` | ROS2 包 `wuji_hand2_beta2_description` |
| `hand/body/package.xml` | ROS2 包 `wuji_description` |
| `hand/attachment/unitree-g1-attachment/` | Unitree G1 适配器 |

---

*End of research distillation.*
