# OSMO · AnySkin · ReSkin — 触觉传感研究蒸馏

> **Date**: 2026-08-16
> **Researcher**: Codex (deepresearch-distill), PaxonHuang review pending
> **Reality**: ✅ 已实现/源码验证 · 🟡 工程可实现 · 🔬 需研发验证 · 🌌 长期方向
> **溯源**: repos at `third-party-deepresearch/repo/{osmo_tactile_glove,anyskin,reskin_sensor}`

---

## 0. 源仓库索引

| 项目 | 路径 | Commit | 论文 | 许可 |
|------|------|--------|------|------|
| OSMO | `repo/osmo_tactile_glove/` | `bfc7328` | [arxiv:2512.08920](https://arxiv.org/abs/2512.08920) | MIT |
| AnySkin | `repo/anyskin/` | `cb13b5b` | [arxiv:2409.08276](https://arxiv.org/abs/2409.08276) | MIT |
| ReSkin | `repo/reskin_sensor/` | `b82de2a` | [arxiv:2111.00071](https://arxiv.org/abs/2111.00071) | MIT |

---

## 1. 核心算法/公式

### 1.1 磁触觉传感原理（三项目共通） ✅

三项目均基于**磁触觉**原理：弹性体内嵌磁体，外部施力导致磁体位移，磁力计（MLX/LSM 系列）检测磁场变化，通过 ML 解码为接触力/纹理。

$$B_i = \sum_{k=1}^{K} \frac{\mu_0}{4\pi} \cdot \frac{3(\mathbf{m}_k \cdot \hat{r}_{ik})\hat{r}_{ik} - \mathbf{m}_k}{|\mathbf{r}_i - \mathbf{r}_k|^3}$$

- $B_i$: 第 i 个磁力计读数（3 轴）；$\mathbf{m}_k$: 第 k 个磁体磁矩；$\mathbf{r}$: 位移向量
- 实际中不解析求解此式，而是用 ML（MLP/CNN）直接从磁场映射到力/滑动/纹理

### 1.2 分位数归一化（OSMO） ✅

OSMO 对触觉通道使用分位数归一化，防止极端值主导：

$$x' = \text{clip}\left(\frac{x - x_{0.02}}{x_{0.98} - x_{0.02}},\; -1.5,\; 1.5\right)$$

- $x_{0.02}$, $x_{0.98}$: 训练数据的 2% 和 98% 分位数
- 来源: OSMO 论文 §4.2；代码在 `glovedp/dp/policy.py` 数据预处理中

### 1.3 预训练/SSL 适配（AnySkin） 🟡

AnySkin 的跨实例迁移方案：
1. **预训练**: 在多个 AnySkin 实例上用自监督学习（SSL）训练编码器
2. **适配**: 新实例仅需少量数据微调
3. **结果**: 跨实例策略迁移仅 -13%（ReSkin -43%），证明跨实例一致性 > 绝对精度
- 来源: AnySkin 论文 §5；代码库中预训练脚本未见（待人工核对）

### 1.4 OSMO diffusion policy 架构 ✅

OSMO 使用 diffusion policy 作为动作解码器：

```
触觉(12 taxel) + 视觉(2D keypoints) → diffusion policy → 机器人关节角
```

- 网络结构: `glovedp/dp/models/cond_unet.py` — 条件 U-Net
- 策略入口: `glovedp/dp/policy.py`
- 关键消融: 纯视觉 55.75% vs +触觉 71.69%（接触密集任务）

---

## 2. 数据结构/通信协议

### 2.1 AnySkin / ReSkin 串行协议（共用） ✅

两个项目共享几乎相同的串行通信协议：

```text
Frame format (burst binary mode):
┌────────────────────────────────────────────────────────┐
│ float[0]  float[1]  float[2]  float[3]  ...  \r\n      │
│ (Bx)      (By)      (Bz)      (Temp)    ...  (terminator)│
└────────────────────────────────────────────────────────┘
  每个磁力计 = 4 × float32 (4 bytes each) = 16 bytes
  总长度 = 4 * num_mags * 4 + 2 bytes (\r\n)
```

| 参数 | AnySkin | ReSkin |
|------|---------|--------|
| 波特率 | 115200 | 115200 |
| 编码 | binary burst (`struct.unpack`) 或 ASCII | binary burst (`struct.unpack`) 或 ASCII |
| 终止符 | `\r\n` | `\r\n` |
| 每磁力计浮点数 | 4 (Bx, By, Bz, Temp) | 4 (Bx, By, Bz, Temp) |
| 温度过滤 | `temp_filtered=True` → 去除每 4 个浮点中的第 4 个 | 同 |
| 设备 ID | `device_id` 字段 | `device_id` 字段 |
| Python 库 | `anyskin` (pip) | `reskin_sensor` (pip) |

**关键 API 签名**：

```python
class AnySkinBase(serial.Serial):
    def __init__(self, num_mags: int = 1, port: str = None,
                 device_id: int = -1, temp_filtered: bool = True,
                 burst_mode: bool = True, baudrate: int = 115200) -> None
    def get_sample(self) -> tuple[float, np.ndarray]  # (timestamp, 3*num_mags array)
    def get_data(self, num_samples: int) -> list[np.ndarray]

class AnySkinProcess:  # 非阻塞后台采集
    def __init__(self, num_mags, port, **kwargs)
    def start() / def stop()
    def get_data(num_samples) -> list[np.ndarray]  # 从 buffer 读取
```

来源: `repo/anyskin/anyskin/sensor.py:25-80`, `repo/reskin_sensor/reskin_sensor/sensor.py:18-60`

### 2.2 OSMO (Bowie) 手套数据协议 ✅

OSMO 内部代号 "Bowie"，通过串口读取 12 个磁触觉 taxel + IMU 四元数：

```python
class BowieGlove:  # repo/osmo_tactile_glove/labs/glove2robot/utils/bowie.py
    def read(self) -> dict  # 返回 40 个磁场值 (5 taxel × 8 channels) + 20 个四元数分量 (5 IMU × 4)

class BowieGloveStream:  # glove_utils.py — 线程化流式读取
    def __init__(self, port)
    def read_data(self, stop_event)  # 后台线程持续读取
    # raw_data = queue.Queue()  # (timestamp_ns, data) 对
    # bowie_mag_deque = [deque(maxlen=10) for _ in range(40)]   # 5 taxel × 8 channels
    # bowie_quat_deque = [deque(maxlen=10) for _ in range(20)]   # 5 IMU × 4 quat
```

- 设备发现: `port.product == "BowieGlove"`（USB CDC 产品名）
- 数据流: 串口 → `BowieGlove.read()` → queue → diffusion policy 预处理
- 采样率: 论文称 100Hz（待人工核对，代码中未见显式设定）

来源: `repo/osmo_tactile_glove/labs/glove2robot/utils/glove_utils.py:20-50`

### 2.3 OSMO 数据采集与重定向管线 ✅

```
OSMO 手套 → 串口 → BowieGlove.read() → raw_data queue
                                          ↓
                         HaMeR 手部关键点提取 (postprocess/extract_hamer.py)
                                          ↓
                         construct_retarget_dataset.py → Psyonic Ability Hand 关节角
                                          ↓
                         diffusion policy (glovedp/dp/) → 机器人动作
```

关键文件:
- `labs/glove2robot/postprocess/extract_hamer.py`: HaMeR → 3D 手部关键点
- `labs/glove2robot/utils/hamer_utils.py`: HaMeR 集成
- `kinematics/construct_retarget_dataset.py`: 关键点 → Psyonic 关节角（重定向）
- `glovedp/dp/policy.py`: diffusion policy 训练/推理

---

## 3. 硬件拓扑

### 3.1 AnySkin 硬件拓扑 ✅

```
┌─────────────────────────────────────────────┐
│              AnySkin 传感器                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │ Magnet 1 │  │ Magnet 2 │  │ Magnet N │    │
│  │ (MQFP-15-│  │ (MQFP-15-│  │ (25μm)   │    │
│  │  7,25μm) │  │  7,25μm) │  │          │    │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘    │
│       │              │              │          │
│       ▼              ▼              ▼          │
│  ┌──────────────────────────────────────┐      │
│  │    Magnetometer array (I2C)          │      │
│  │    (MLX90393 / LSM6DSO)              │      │
│  └──────────────┬───────────────────────┘      │
│                 │                               │
│         QWIIC (I2C @ 400kHz)                    │
│                 │                               │
│  ┌──────────────▼──────────────────────┐       │
│  │    Adafruit QT Py (SAMD21)          │       │
│  │    USB-CDC @ 115200 baud             │       │
│  └──────────────┬──────────────────────┘       │
└─────────────────┼──────────────────────────────┘
                  │ USB-C
                  ▼
            Host PC (Python anyskin library)
```

- 磁体: MQFP-15-7 (25μm 颗粒)，固化后磁化（非固化中磁场）
- 自对准自粘：免胶水/螺丝，12s 更换，可复用
- 跨实例 std: 0.12（vs ReSkin 0.54）
- 来源: AnySkin 论文 §3；`repo/anyskin/README.md`

### 3.2 ReSkin 硬件拓扑 ✅

```
┌─────────────────────────────────────────────┐
│              ReSkin 5X board                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │ Mag 1    │  │ Mag 2    │  │ Mag 5    │    │
│  │(neodymium│  │(neodymium│  │(neodymium│    │
│  │ disc)    │  │ disc)    │  │ disc)    │    │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘    │
│       │              │              │          │
│       ▼              ▼              ▼          │
│  ┌──────────────────────────────────────┐      │
│  │    5× MLX90393 (I2C @ 0x0C..0x10)   │      │
│  └──────────────┬───────────────────────┘      │
│                 │                               │
│         I2C @ 400kHz                            │
│                 │                               │
│  ┌──────────────▼──────────────────────┐       │
│  │    Adafruit Trinket M0 / QT Py      │       │
│  │    USB-CDC @ 115200 baud             │       │
│  └──────────────┬──────────────────────┘       │
└─────────────────┼──────────────────────────────┘
                  │ USB-C
                  ▼
            Host PC (Python reskin_sensor library)
```

- 磁体: neodymium disc（比 AnySkin 的 MQFP 颗粒大）
- 封装: 5X board（5 磁力计排列）
- 跨实例一致性: 较差（std 0.54 vs AnySkin 0.12）
- 来源: ReSkin 论文 §3；`repo/reskin_sensor/README.md`

### 3.3 OSMO 硬件拓扑 ✅

```
┌─────────────────────────────────────────────┐
│              OSMO 触觉手套                    │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐│
│  │Taxel1│ │Taxel2│ │Taxel3│ │Taxel4│ │Taxel5││
│  │(拇) │ │(食) │ │(中) │ │(无) │ │(小) ││
│  │3-axis│ │3-axis│ │3-axis│ │3-axis│ │3-axis││
│  │mag   │ │mag   │ │mag   │ │mag   │ │mag   ││
│  └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘│
│     │         │        │        │        │    │
│  ┌──▼─────────▼────────▼────────▼────────▼──┐│
│  │  12 taxel (5 finger × ~2.4 ch)            ││
│  │  + 5 IMU (每指一个) → 20 quat values       ││
│  │  USB-CDC serial → "BowieGlove" device      ││
│  └──────────────────┬────────────────────────┘│
└─────────────────────┼────────────────────────┘
                      │ USB-C
                      ▼
          Host PC (Python bowie/glovedp)
```

- 12 taxel: 5 个指尖磁触觉传感器（每个 3 轴磁场 + 可能多通道）
- 5 IMU: 每指一个（与 FSGlove 类似但更少 DoF）
- 人机同手套: 同一 OSMO 手套既用于人类数据采集又用于机器人触觉反馈
- 来源: OSMO 论文 §3；代码 `bowie.py` + `glove_utils.py`

---

## 4. 关键 API 签名

### 4.1 AnySkin ✅

```python
# 阻塞式采集
from anyskin import AnySkinBase
sensor = AnySkinBase(num_mags=5, port="/dev/ttyACM0", baudrate=115200,
                     burst_mode=True, temp_filtered=True)
timestamp, data = sensor.get_sample()  # data: np.ndarray, shape=(3*num_mags,)

# 非阻塞后台采集
from anyskin import AnySkinProcess
proc = AnySkinProcess(num_mags=5, port="/dev/ttyACM0", buffer_size=500)
proc.start()
data_list = proc.get_data(num_samples=10)

# 可视化
from anyskin.visualizations import anyskin_viz
anyskin_viz.run(port="/dev/ttyACM0")
```

### 4.2 ReSkin ✅

```python
from reskin_sensor import ReSkinBase, ReSkinData
sensor = ReSkinBase(num_mags=5, port="/dev/ttyACM0", baudrate=115200,
                    burst_mode=True, temp_filtered=False,
                    reskin_data_struct=True)
# 返回 ReSkinData namedtuple:
# ReSkinData(time, acq_delay, data, dev_id)
sample: ReSkinData = sensor.get_sample()
```

### 4.3 OSMO (Bowie) ✅

```python
from glove2robot.utils.bowie import BowieGlove
from glove2robot.utils.glove_utils import BowieGloveStream, find_bowie_port

# 发现设备
port = find_bowie_port()  # 遍历 USB CDC,匹配 product == "BowieGlove"

# 阻塞式
glove = BowieGlove(port)
data = glove.read()  # dict: {mag: [...40...], quat: [...20...]}

# 流式
stream = BowieGloveStream(port)
stream.read_data(stop_event)  # 后台线程,数据入 queue
```

---

## 5. 工程踩坑记录

| 坑 | 来源 | 为什么 | 修复 |
|----|------|--------|------|
| 串口 buffer 溢出导致脏数据 | AnySkin/ReSkin | `in_waiting > 4000` 时 stale data 累积 | `reset_input_buffer()` + 重新同步到 `\r\n` |
| 温度通道干扰 | AnySkin/ReSkin | 每 4 个 float 中第 4 个是温度，非力数据 | `temp_filtered=True` → mask 掉每 4 个中的第 4 个 |
| 磁触觉对铁磁物敏感 | ReSkin/AnySkin | 环境铁磁物体改变磁场分布 | 远离铁磁物；AnySkin 用 MQFP 颗粒减少此问题 |
| 穿戴导致基线漂移 | AnySkin | 传感器佩戴后零点偏移 | 按 `B` 键重新校准零点（`anyskin_viz` 内置） |
| 连续触觉误导操作者 | OSMO | 滑瓶实验：连续触觉让操作者误判表面特性 | >100g 切断触觉只留力反馈（OSMO 论文 §6.3） |
| 纯视觉在接触密集任务失败 | OSMO | 纯视觉 55.75% vs +触觉 71.69% | 必须融合触觉通道 |
| OSMO 代码中 "bowie" 代号 | OSMO | 内部代号未更新 | Bowie = OSMO，可互换理解 |
| AnySkin 与 ReSkin API 几乎相同 | 两者 | AnySkin 从 ReSkin fork 改进 | 迁移成本低，但数据格式不完全兼容（temp_filtered 默认值不同） |

---

## 6. 三源交叉对比表

| 维度 | OSMO | AnySkin | ReSkin |
|------|------|---------|--------|
| **方法** | 磁触觉 + IMU + diffusion policy | 磁触觉（MQFP 颗粒）+ SSL | 磁触觉（钕磁体）+ ML 解码 |
| **cost** | 中（12 taxel + 5 IMU） | 低（startup kit） | 低（5X board） |
| **分辨率** | 12 taxel, 5 指尖 | 1-15 磁力计/指尖 | 5 磁力计/板 |
| **跨实例一致性** | — | std 0.12 ✅ | std 0.54 ❌ |
| **策略迁移损失** | — | -13% ✅ | -43% ❌ |
| **ML 依赖** | diffusion policy（重） | SSL 预训练 + 微调 | MLP/CNN（轻） |
| **人机同手套** | ✅（核心创新） | ❌ | ❌ |
| **采样率** | ~100Hz (待人工核对) | 依赖串口/MCU | 依赖串口/MCU |
| **通信** | USB-CDC serial | USB-CDC serial @115200 | USB-CDC serial @115200 |
| **Python 库** | 内部（bowie/glovedp） | `anyskin` (pip) | `reskin_sensor` (pip) |
| **EgoGlove 相关度** | 🔴 高（人机同手套理念） | 🟡 中高（若上触觉） | 🟡 中（基座参考） |

---

## 7. 对 EgoGlove 主线的启示

### ✅ 精华（吸收）

| 决策点 | 来源 | 落点 |
|--------|------|------|
| 触觉升级首选 AnySkin 式磁触觉 | AnySkin | EgoGlove Pro 触觉 roadmap；跨实例一致性是规模化前提 |
| 人机同手套理念 | OSMO | 手套既做人类数据采集又做机器人触觉反馈——降低系统复杂度 |
| 分位数归一化作为触觉标准预处理 | OSMO | 数据管道预处理标准 |
| diffusion policy + 触觉通道 | OSMO | L2/L3 动作解码器候选（触觉通道接入后） |
| 串口协议结构（4 float/mag + \r\n） | AnySkin/ReSkin | 触觉传感器接入参考协议格式 |

### ❌ 糟粕/风险（丢弃或标注）

| 风险 | 来源 | 处理 |
|------|------|------|
| 纯视觉在接触密集任务失败 | OSMO | 佐证"L3 视觉需触觉补足"；非否定视觉 |
| 磁触觉对铁磁物敏感 | ReSkin/AnySkin | 触觉硬件选型限制条件 |
| 连续触觉误导操作者 | OSMO | 力反馈设计需考虑阈值切断 |
| ReSkin 跨实例一致性差 | ReSkin | 不用 ReSkin 方案，选 AnySkin |
| diffusion policy 计算量大 | OSMO | 边缘部署需量化；🌌 长期方向 |

---

## 8. 溯源表

| 项目 | arXiv | 仓库 | Commit |
|------|-------|------|--------|
| OSMO | [2512.08920](https://arxiv.org/abs/2512.08920) | `repo/osmo_tactile_glove/` | `bfc7328` |
| AnySkin | [2409.08276](https://arxiv.org/abs/2409.08276) | `repo/anyskin/` | `cb13b5b` |
| ReSkin | [2111.00071](https://arxiv.org/abs/2111.00071) | `repo/reskin_sensor/` | `b82de2a` |
