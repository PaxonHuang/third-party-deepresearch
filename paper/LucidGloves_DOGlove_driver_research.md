# LucidGloves · OpenGloves-Driver · DOGlove — 研究蒸馏

> 为 EgoGlove 项目提炼 6 个月后仍有用的知识。distill ≠ translate。

**生成日期**: 2026-08-16
**真实性标注**: ✅ 已实现/验证 · 🟡 工程可实现(6–12 月) · 🔬 需研发验证 · 🌌 长期方向 · 待人工核对

---

## 目录
1. [源仓库索引](#1-源仓库索引)
2. [LucidGloves 固件](#2-lucidgloves-固件)
3. [OpenGloves-Driver (SteamVR 驱动)](#3-opengloves-driver-steamvr-驱动)
4. [DOGlove (力反馈手套)](#4-doglove-力反馈手套)
5. [三源交叉对比表](#5-三源交叉对比表)
6. [对 EgoGlove 的启示](#6-对-egoglove-的启示)

---

## 1. 源仓库索引

| 项目 | 路径 | Commit | 上游 URL | 许可 |
|------|------|--------|----------|------|
| lucidgloves | `repo/lucidgloves/` | `76472c70887689ae97e4fb4f5a976eb6d715a191` | https://github.com/LucidVR/lucidgloves | GPL-3.0 |
| opengloves-driver | `repo/opengloves-driver/` (branch develop) | `9e1f2fdcd8994ed21089172630e33d6ed31f8710` | https://github.com/LucidVR/opengloves-driver | MIT |
| DOGlove | `repo/DOGlove-main/DOGlove-main/` | 无 .git (ZIP 解压, 含 :Zone.Identifier) | https://github.com/doublehan07/DOGlove | MIT |

**关键文件速查**:

```text
lucidgloves/
├── Config.h                          # 引脚映射 + 编码/通信模式
├── AdvancedConfig.h                  # 环路时间、滤波、ANALOG_MAX
├── src/Main.cpp                      # 主循环: 采集→编码→通信→力反馈
├── src/Util/ConfigUtils.h            # MUX()/COMM_*宏
├── src/Util/DataStructs.h            # OutboundData/DecodedData 结构
├── src/Controller/InputManager.cpp   # 校准数学 + sin/cos 混合
├── src/Controller/Haptics.cpp        # 舵机力反馈缩放
├── src/Controller/Gesture.cpp        # 抓取/捏合/扳机手势
├── src/Encoding/AlphaEncoding.cpp    # Alpha 协议编解码
├── src/Encoding/LegacyEncoding.cpp   # 旧版 & 分隔符协议
├── src/Communication/SerialCommunication.cpp
└── src/Communication/BTSerialCommunication.cpp

opengloves-driver/
├── server/include/opengloves_interface.h    # og::Input/Output/IDevice/Server 核心接口
├── server/src/communication/encoding/alpha_encoding_service.cpp  # PC 端 Alpha 解析
├── server/src/communication/managers/hardware_communication_manager.cpp
├── server/src/communication/services/service_serial_win.cpp       # Win32 串口
├── server/src/communication/services/service_bluetooth_win.cpp    # Win32 BT (ws2bth)
├── server/src/communication/probers/prober_serial_identifiers_win.cpp
├── server/src/device/lucidgloves/lucidgloves_device.cpp
├── server/src/device/lucidgloves/discovery/lucidgloves_fw_discovery.cpp
├── driver/src/device/providers/physical_device_provider.cpp       # SteamVR 入口
├── driver/src/device/drivers/knuckle_device_driver.h              # Index Knuckles 映射
├── driver/src/device/pose/pose_calibration.cpp                    # 姿态校准数学
└── driver/src/util/driver_math.h                                  # 四元数/矩阵工具

DOGlove-main/
├── glove_mcu.py          # 旋转编码器 MCU UART 读取 → UDP
├── servo.py              # Dynamixel 舵机位置读取 → UDP
├── uart_reader_base.py   # 线程化 UART 读取基类
├── udp_receiver.py       # UDP 接收 + URDF 关节变换
├── fk.py                 # MuJoCo FK 可视化
├── fk_ik_core.py         # FK→多机械手 IK 引擎 (mink)
├── tracker.py            # Vive Tracker (SteamVR) 姿态
├── tools/udp_record.py   # 硬件数据包录制 (JSONL)
└── tools/udp_replay.py   # 录制回放
```

---

## 2. LucidGloves 固件

> **一句话**: 弦驱霍尔/电位计 → ESP32 ADC → min/max 动态校准 → Alpha 文本协议 → 串口/BT。力反馈用舵机。

### 2.1 核心算法/公式

#### 2.1.1 弦驱动 + 旋转霍尔传感器原理 ✅

LucidGloves 机械上用**鲍登线 (bowden cable)** 连接指尖到指根的旋转 spool。spool 轴上装 **旋转霍尔传感器 (rotary Hall effect, 双通道 sin/cos)** 或**电位计**。手指弯曲拉动线 → spool 旋转 → 传感器输出角度变化。

- **电位计模式** (`MIXING_NONE`): 单通道线性, 输出直接正比于角度。
- **双霍尔模式** (`MIXING_SINCOS`): 两路正交信号 (sin, cos), 需要 atan2 解算 + 旋转计数。

来源: `Config.h:118-121` FLEXION_MIXING, `InputManager.cpp:67-104` sinCosMix()。

#### 2.1.2 sin/cos 混合解算 (旋转霍尔) ✅

`InputManager.cpp` `sinCosMix()`:

```cpp
int sinScaled = map(sinRaw, sinMin[i], sinMax[i], -ANALOG_MAX, ANALOG_MAX);
int cosScaled = map(cosRaw, cosMin[i], cosMax[i], -ANALOG_MAX, ANALOG_MAX);
double angleRaw = atan2(sinScaled, cosScaled);
// 旋转计数: 检测象限跳变
if (((angleRaw > 0) != atanPositive[i]) && sinScaled > cosScaled) {
    totalOffset1[i] += atanPositive[i] ? 1 : -1;
}
double totalAngle = angleRaw + 2*PI * totalOffset1[i];
```

数学表达:

```latex
\theta_{raw} = \text{atan2}(\sin_{scaled}, \cos_{scaled}) \\
\theta_{total} = \theta_{raw} + 2\pi \cdot N_{rotations}
```

其中 $N_{rotations}$ 通过象限跳变累计: 当 atan2 符号翻转且 sin > cos 时 ±1。

**关键中间校准值** (sinMin/sinMax/cosMin/cosMax) 持久化到 EEPROM, 地址 `0x29` 起 (`saveIntermediate()`)。

#### 2.1.3 动态 min/max 校准 ✅

`InputManager.cpp:111-145` — 持续跟踪每个传感器通道的 max/min:

```latex
\text{pos}_i = \text{map}(\text{raw}_i, \min_i, \max_i, 0, \text{ANALOG\_MAX})
```

- `CALIBRATION_LOOPS = -1` → 永久校准 (持续更新 min/max)。✅ 默认值
- `CALIBRATION_LOOPS = N` → 前 N 个 loop 校准后冻结。
- PIN_CALIB 按钮可重置 `loops = 0` 强制重新校准。
- **Travel 限制** (`saveTravel()`): 将 `(max - min)` 存 EEPROM, 防止校准过冲。当 `max - min > maxTravel` 时钳位另一端。

**EEPROM 布局**:
```text
0x00: flags (bit0 = travel saved, bit1 = intermediate saved)
0x01: maxTravel[10] × int32 (40 bytes)
0x29: sin/cos min/max[5] × 4 × int32 (80 bytes)
```

#### 2.1.4 力反馈舵机缩放 ✅

`Haptics.cpp` `scaleLimits()`:

```latex
\theta_{servo} = \frac{\text{hapticLimit}}{1000} \times 180°
```

输入值 0–1000 → 舵机角度 0–180°。`FLIP_FORCE_FEEDBACK` 反转方向。负值 (`< 0`) 跳过该指 (不写舵机)。

#### 2.1.5 手势判定 ✅

`Gesture.cpp` — 基于平均弯曲度的二值阈值:

```latex
\text{grab} = \frac{\sum_{i \in \{pinky,ring,middle,index\}} \text{flex}_i}{4} > \frac{\text{ANALOG\_MAX}}{2} \\
\text{pinch} = \frac{\text{flex}_{index} + \text{flex}_{thumb}}{2} > \frac{\text{ANALOG\_MAX}}{2} \\
\text{trigger} = \text{flex}_{index} > \frac{\text{ANALOG\_MAX}}{2}
```

### 2.2 数据结构/通信协议

#### 2.2.1 Alpha 编码 (文本协议) ✅

`AlphaEncoding.cpp` `encode()` — **出站帧** (glove → PC):

```text
A<thumb>B<index>C<middle>D<ring>E<pinky>F<joyX>G<joyY>P<triggerVal>H?I?J?K?L?M?N?O?(AB)<tsplay>(BB)<isplay>...(EB)<psplay>\n
```

| 字段 | 字符 | 范围 | 含义 |
|------|------|------|------|
| Thumb curl | A | 0–ANALOG_MAX | 拇指弯曲 |
| Index curl | B | 0–ANALOG_MAX | 食指弯曲 |
| Middle curl | C | 0–ANALOG_MAX | 中指弯曲 |
| Ring curl | D | 0–ANALOG_MAX | 无名指弯曲 |
| Pinky curl | E | 0–ANALOG_MAX | 小指弯曲 |
| Joystick X | F | 0–ANALOG_MAX | 摇杆 X |
| Joystick Y | G | 0–ANALOG_MAX | 摇杆 Y |
| Trigger | P | 0–ANALOG_MAX | 模拟扳机 (index - ANALOG_MAX/2)*2 |
| JoyClick | H | flag | 摇杆按下 (存在即真) |
| Trigger btn | I | flag | |
| A btn | J | flag | |
| B btn | K | flag | |
| Grab | L | flag | 抓取手势 |
| Pinch | M | flag | 捏合手势 |
| Menu | N | flag | |
| Calib | O | flag | 校准中 |
| Thumb splay | (AB) | 0–ANALOG_MAX | 拇指侧摆 (可选) |
| Index splay | (BB) | | |
| ... | (CB)(DB)(EB) | | |

- 每帧以 `\n` 结尾。
- 布尔字段**存在即 true**, 无值后缀。
- Trigger 派生: `(fingers[1] > ANALOG_MAX/2) ? (fingers[1] - ANALOG_MAX/2) * 2 : 0`。
- `USING_SPLAY=false` 时 splay 段省略。

**入站帧** (PC → glove, 力反馈):

```text
A<thumbFF>B<indexFF>C<middleFF>D<ringFF>E<pinkyFF>\n
```

或特殊命令 (含 `Z` 字符):
```text
ZSaveInter\n    # 保存 sin/cos 中间校准
ZSaveTravel\n   # 保存 travel 限制
ZClearData\n    # 清除 EEPROM flags
```

**解析** `decodeData()`: 遍历 A–E 找数字, `Z` 触发命令匹配。

#### 2.2.2 Legacy 编码 ✅

`LegacyEncoding.cpp` — `&` 分隔的纯数字:

```text
<thumb>&<index>&<middle>&<ring>&<pinky>&<joyX>&<joyY>&<joyClick>&<triggerBtn>&<aBtn>&<bBtn>&<grab>&<pinch>\n
```

无字段标签, 靠顺序解析。已被 Alpha 取代。

#### 2.2.3 OutboundData 结构 ✅

`DataStructs.h`:

```c
struct OutboundData {
    int fingers[5];        // curl, 0–ANALOG_MAX
    int joyX, joyY;
    bool joyClick, triggerButton, aButton, bButton, grab, pinch, calib, menu;
    #if USING_SPLAY
    int splay[5];           // splay, 0–ANALOG_MAX
    #endif
};

struct DecodedData {
    ReceivedFields fields;  // 哪些字段收到
    int servoValues[5];     // 力反馈值
    const char* command;    // 特殊命令
};
```

### 2.3 硬件拓扑

#### ESP32 (DOIT V1) 引脚映射 ✅ (`Config.h:62-96`)

```text
                ESP32 DOIT V1
    ┌──────────────────────────────────┐
    │ GPIO32 ──┐ pinky flex (ADC1)     │
    │ GPIO35 ──┤ ring flex   (ADC1)    │── ADC1: 无 WiFi 干扰
    │ GPIO34 ──┤ middle flex (ADC1)    │
    │ GPIO39 ──┤ index flex  (ADC1)    │
    │ GPIO36 ──┤ thumb flex  (ADC1)    │
    │ GPIO33 ──┤ joy X       (ADC1)    │
    │ GPIO25 ──┤ joy Y       (ADC2*)   │  *ADC2 与 WiFi 互斥
    │ GPIO26 ──┘ joy button (INPUT_PULLUP)
    │                                  │
    │ GPIO27 ──┐ A button              │
    │ GPIO14 ──┤ B button              │
    │ GPIO23 ──┘ pinch button          │
    │ GPIO34 ─── menu button (与 middle 共用!)
    │ GPIO15 ─── calib button          │
    │ GPIO2  ─── debug LED             │
    │                                  │
    │ GPIO19 ──┐ pinky motor (PWM)     │── 舵机力反馈
    │ GPIO18 ──┤ ring motor            │
    │ GPIO5  ──┤ middle motor          │
    │ GPIO17 ──┤ index motor           │
    │ GPIO16 ──┘ thumb motor           │
    │                                  │
    │ GPIO27,14,12,13 ── MUX select    │── 74HC4067 16:1 多路复用
    │ GPIO35 ────────── MUX input      │  (用于 splay/second 通道)
    └──────────────────────────────────┘
```

**关键约束**:
- **ANALOG_MAX = 4095** (ESP32 12-bit ADC), **1023** (AVR 10-bit)。
- ADC1 (GPIO32-39) 与 WiFi 不冲突; ADC2 (GPIO25/26 等) 在 WiFi 开启时不可用。
- **MUX 宏**: `MUX(p) = p + 100`, `UNMUX(p) = p % 100`, `ISMUX(p) = p >= 100`。多路复用延迟 `5µs` (`MULTIPLEXER_DELAY`)。
- 串口波特率 115200; BT 名称 `lucidgloves-left`。
- 舵机用 ESP32Servo 库 (PWM, 非标准 I2C)。

#### DOF 结构

```text
          Curl (5)          Splay (5, 可选)
       ┌─→ ADC ─→ min/max ─→ map(0..4095)
spool ─┤
       └─→ [splay] MUX ─→ ADC ─→ min/max ─→ map
```

- 默认 **5 DOF curl** (每指一个弯曲值)。
- `USING_SPLAY=true` 时 +5 DOF splay (每指侧摆), 通过 MUX 扩展。
- 无关节级独立 (单 curl 值代表整指弯曲)。

### 2.4 关键 API 签名

```cpp
// InputManager — 采集 + 校准
void getFingerPositions(bool calibrating, bool reset, int* fingerPos);
  // calibrating: 是否更新 min/max
  // reset: 是否重置 min/max 到极值
  // fingerPos: 输出 5 个 curl 值 [0, ANALOG_MAX]
  // 前置: setupInputs() 已调用

int sinCosMix(int sinPin, int cosPin, int i);
  // 返回 totalAngle * ANALOG_MAX (int)
  // i: 手指索引 0-4, 用于 sin/cos 校准数组

// IEncoding
void encode(OutboundData data, char* stringToEncode);
DecodedData decodeData(char* stringToDecode);

// ICommunication
void start();
void output(char* data);          // 发送编码帧
bool readData(char* input);        // 读取入站帧 (阻塞直到 \n)

// Haptics
void writeServoHaptics(int* hapticLimits);  // 5 个 0-1000 值
```

### 2.5 工程陷阱

1. **ADC2 与 WiFi 互斥** 🟡: ESP32 上 `COMM_WIFISERIAL` 模式下 ADC2 引脚 (GPIO25 等) 读取失败。默认 Config.h 把 joyY 放在 GPIO25 — WiFi 模式下需迁移到 ADC1 引脚。
2. **GPIO34 被 menu 和 middle flex 共用** ✅: `PIN_MIDDLE = 34` 且 `PIN_MENU_BTN = 34`。实际不能同时用。
3. **fingerPos 数组大小 10 但只填 5** ✅: `Main.cpp` 复制 10 个值但 `OutboundData.fingers[5]` — splay 存在时另 5 个进 `data.splay`, 不存在时静默丢弃。
4. **校准跳变** 🟡: 永久校准模式下 min/max 持续更新。如果传感器瞬时噪声超出正常范围, max/min 被拉宽, 后续 map 范围变大导致分辨率下降。Travel 限制是缓解方案但需手动 `SaveTravel`。
5. **Alpha 编码 trigger 派生 hack** ✅: `AlphaEncoding.cpp:7` — trigger 从 `fingers[1]` (index) 派生而非独立字段。注释说是 v0.5 驱动的临时修复。
6. **sin/cos 象限跳变检测脆弱** 🟡: `sinScaled > cosScaled` 条件在噪声下可能误计旋转数。RunningMedian 滤波可选但默认关闭 (`ENABLE_MEDIAN_FILTER=false`)。
7. **EEPROM 地址碰撞风险** 🟡: Travel 存 0x01–0x28 (10×4=40B), Intermediate 存 0x29 起。ESP32 EEPROM 大小 `0x78 + 1 = 121B`, 5 指 sin/cos 4×4=16×4=80B, 到 0x79 刚好。留余量极小。
8. **Legacy 编码无字段标签** ✅: 顺序解析, 任何字段数变化都会错位。

---

## 3. OpenGloves-Driver (SteamVR 驱动)

> **一句话**: C++ SteamVR 驱动 = server (设备发现+串口/BT 通信+Alpha 解析) + driver (SteamVR 注册+Knuckles 映射+姿态校准)。通过 named pipe 桥接。

### 3.1 核心算法/公式

#### 3.1.1 Alpha 编码解析 (PC 端) ✅

`alpha_encoding_service.cpp` — **键值映射表** (`alpha_encoding_input_key_strings`):

```text
"A"   → ThumbCurl        "(AB)" → ThumbSplay
"B"   → IndexCurl        "(BB)" → IndexSplay
...
"(AAA)" → ThumbJoint0     # 4 关节/指 (未在固件端实现)
"(AAB)" → ThumbJoint1
...
"F"   → Joystick_X        "G" → Joystick_Y
"H"   → JoyClick          "I" → Trigger_Click
"J"   → A_Click           "K" → B_Click
"L"   → Grab_Gesture      "M" → Pinch_Gesture
"N"   → Menu_Click        "O" → Calibration_Click
"P"   → Trigger_Value
"Z"   → Info              "(ZV)" → FW version
"(ZG)" → DeviceType       "(ZH)" → Hand
```

**解析算法** `ParseToMap()`:
1. 遍历字符串, 检测 `(` 开头的多字符键 (`(AB)` 等) 或单字符键 (`A`–`P`)。
2. 键后的连续数字为值 (`std::stof`)。
3. 布尔键 (H–O) 存在即 true。
4. `Z` 键存在 → Info 包, 否则 Peripheral 包。

**归一化**: `result.flexion[finger][curl] = stof / max_analog_value` (默认 4095) → float [0, 1]。

#### 3.1.2 SteamVR 姿态校准数学 ✅

`pose_calibration.cpp` `CompleteCalibration()`:

```latex
q_{offset} = -q_{controller} \cdot q_{maintain} \\
\vec{p}_{offset} = (\vec{p}_{maintain} - \vec{p}_{controller}) \cdot (-q_{controller})
```

- `maintain_pose`: 校准开始时手套的姿态 (静止)。
- `controller_pose`: 参考控制器 (如 Index 手柄) 当前姿态。
- 偏移 = 参考控制器到手套的变换。应用时: `pose = offset × controller_pose`。

#### 3.1.3 设备发现/轮询循环 ✅

`lucidgloves_fw_discovery.cpp`:

```text
StartDiscovery() → 为每个 device_config 启动 ProberThread
ProberThread: while is_active_ { prober->InquireDevices(); sleep(2s) }
OnDeviceFound: service → AlphaEncodingService → HardwareCommunicationManager → LucidglovesDevice → callback_(device)
```

- 串口探测: VID/PID 匹配 (SetupAPI) 或指定 port_name。
- 蓝牙探测: 按设备名连接 (ws2bth)。
- `auto_probe` 标记未实现 (`"Auto probing is currently not implemented."`)。

### 3.2 数据结构/通信协议

#### 3.2.1 核心接口类型 ✅ (`opengloves_interface.h`)

```cpp
namespace og {
  enum Hand { kHandLeft, kHandRight };
  enum DeviceType { kDeviceType_lucidgloves };

  // 输入: 设备 → 驱动
  struct InputPeripheralData {
    std::array<std::array<float, 4>, 5> flexion;  // [finger][joint], 归一化
    std::array<float, 5> splay;
    Button trigger, A, B, menu, calibrate;
    Joystick joystick;
    Gesture grab, pinch;
  };
  struct InputInfoData {
    Hand hand; DeviceType device_type; int firmware_version;
  };
  union InputData { InputInfoData info; InputPeripheralData peripheral; };
  struct Input { InputData data; InputDataType type; };

  // 输出: 驱动 → 设备
  struct OutputForceFeedbackData { int16_t thumb, index, middle, ring, pinky; };
  struct OutputHapticData { float duration, frequency, amplitude; };
  struct Output { OutputDataType type; OutputData data; };

  // 设备接口
  class IDevice {
    virtual DeviceConfiguration GetConfiguration() = 0;
    virtual void ListenForInput(std::function<void(const InputPeripheralData&)> callback) = 0;
    virtual void Output(const Output& output) = 0;
  };
  class IDeviceDiscoverer {
    virtual void StartDiscovery(std::function<void(std::unique_ptr<IDevice>)> callback) = 0;
  };
}
```

**关键**: `flexion[finger][4]` 支持 4 关节/指, 但固件只填 `[finger][0]` (curl)。driver 端结构已预留多关节。

#### 3.2.2 通信线程模型 ✅

`hardware_communication_manager.cpp` `CommunicationThread()`:

```text
while thread_active:
    service->ReceiveNextPacket(received_string)  # 阻塞, 读到 \n
    input = encoding->DecodePacket(received_string)
    callback_(input)
    service->RawWrite(queued_write_string + "\n")  # 拼接输出
```

- 输出帧拼接: 多次 `WriteOutput()` 累积到 `queued_write_string`, 每轮一次性写出 + `\n`。
- Named Pipe 用于 force feedback 输入 (SteamVR → server)。

#### 3.2.3 编码输出帧 ✅

`alpha_encoding_service.cpp` `EncodePacket()`:

```text
# Force Feedback
A<thumb>B<index>C<middle>D<ring>E<pinky>\n

# Haptic
F<freq>G<dur>H<amp>\n

# Info / 控制命令
(ZA)        # start streaming
(ZZ)        # stop streaming
Z           # get info
```

### 3.3 硬件拓扑 (PC 端)

```text
   ┌─ ESP32 Glove ─┐         ┌─ Win32 Serial (COMx, 115200, 8N1) ─┐
   │  Alpha frames │ ──USB──→│  ReadFile until \n                 │
   │  \n-delimited │         │  WaitCommEvent(EV_RXCHAR)          │
   └───────────────┘         └────────────────────────────────────┘
                                     │
   ┌─ ESP32 Glove ─┐         ┌─ Win32 Bluetooth (ws2bth) ──────────┐
   │  BT Serial    │ ──BT───→│  connect by device name             │
   │  SPP          │         │  recv until \n                     │
   └───────────────┘         └────────────────────────────────────┘
                                     │
                                     ▼
                    ┌─ HardwareCommunicationManager ─┐
                    │  CommunicationThread (per device)│
                    │  DecodePacket / EncodePacket     │
                    └──────────────────────────────────┘
                                     │ Named Pipe (force feedback)
                                     ▼
                    ┌─ KnuckleDeviceDriver ────────────┐
                    │  SteamVR ITrackedDeviceDriver     │
                    │  25 components (Knuckles layout)  │
                    │  PoseCalibration (offset math)    │
                    └──────────────────────────────────┘
                                     │
                                     ▼
                              SteamVR Runtime
```

**串口参数** ✅ (`service_serial_win.cpp`): 115200 baud, 8 bit, 1 stop, no parity, DTR enable, 读超时 50ms interval。

### 3.4 关键 API 签名

```cpp
// og::Server — 顶层
Server(ServerConfiguration configuration);
bool StartProber(std::function<void(std::unique_ptr<IDevice>)> callback);
bool StopProber();

// AlphaEncodingService — 编解码
AlphaEncodingService(og::DeviceAlphaEncodingConfiguration config);
  // config.max_analog_value: 归一化分母 (默认 4095)
og::Input DecodePacket(const std::string& buff);
  // 返回 Input{type, data}
  // type = kInputDataType_Peripheral | kInputDataType_Info | kInputDataType_Invalid
std::string EncodePacket(const og::Output& output);

// ICommunicationService — 传输层
bool ReceiveNextPacket(std::string& buff);  // 阻塞到 \n
bool RawWrite(const std::string& buff);
bool IsConnected();
std::string GetIdentifier();

// PhysicalDeviceProvider — SteamVR 入口
vr::EVRInitError Init(vr::IVRDriverContext* pDriverContext);
  // 注册 KnuckleDeviceDriver × 2 (left/right)
  // 启动 og::Server prober
  // 发现设备后 SetDeviceDriver(std::move(found_device))
```

### 3.5 工程陷阱

1. **auto_probe 未实现** ✅: `lucidgloves_fw_discovery.cpp:33` — "Auto probing is currently not implemented."。必须手动配置 port_name 或 BT name。
2. **flexion[4] 多关节预留但固件不填** 🟡: driver 结构 `array<array<float,4>,5>` 支持每指 4 关节, 但 lucidgloves 固件只发 curl (A–E)。多关节键 `(AAA)`–`(EAD)` 在固件端无对应编码。
3. **通信线程无超时退出** 🟡: `CommunicationThread` 在 `ReceiveNextPacket` 返回 false 时直接 return, 但阻塞读取 (WaitCommEvent) 期间 `thread_active_` 变 false 不能立即退出。析构靠 `PrepareDisconnect()` + `CancelIoEx()`。
4. **姿态校准假设静止** ✅: `StartCalibration` 清零速度/角速度, 但不验证实际是否静止。
5. **仅支持 Windows** ✅: serial/bluetooth service 用 `#ifdef _WIN32`, linux 实现文件不存在 (`service_serial_linux.h` 被引用但不在 repo)。
6. **Logger 去重** ✅: `Logger::Log` 比较 `last_message_`, 相同消息跳过。可能导致关键错误被吞 (如连续相同超时)。

---

## 4. DOGlove (力反馈手套)

> **一句话**: STM32 + 旋转磁编码器 (16 通道) → UART 921600 → Python → UDP → MuJoCo FK → 多机械手 IK (mink)。力反馈用 Dynamixel 舵机 + LRA。

### 4.1 核心算法/公式

#### 4.1.1 旋转磁编码器 → 关节角度 ✅

`glove_mcu.py` `process_block()`:

```python
data = struct.unpack("<I16iI", block[4:])  # uint32 + 16×int32 + uint32
reference_voltage = data[0]
data_values = data[1:-1]  # 16 个 ADC 值
crc = data[-1]

voltages = [(float(val) * 5.0 / 0x7FFFFF) for val in data_values]
reference_voltage = reference_voltage / 1000000
joint_angles = [val / reference_voltage * 360 for val in voltages if reference_voltage != 0]
```

```latex
V_{raw} = \frac{\text{ADC}_{code} \times 5.0}{2^{23} - 1} \\
\theta_{deg} = \frac{V_{raw}}{V_{ref}} \times 360°
```

- ADC 分辨率: 24-bit (`0x7FFFFF = 2^23 - 1`), 但实际是 int32 存储的 24 位有效。
- 参考电压以 µV 为单位传输 (`/ 1000000`)。
- 16 通道 = 每指 3 关节 (DIP/PIP/MCP bend) + 1 split = 4×5 = 20, 但实际 16 = 3 bend × 5 + 1 split... 待人工核对 (代码注释未说明 16 通道到 5 指 21 关节的映射)。

**关键**: 这是**电压比值法** — 编码器输出电压与参考电压的比值映射到 360°, 消除供电波动影响。

#### 4.1.2 Dynamixel 舵机位置读取 ✅

`servo.py`:

```python
ADDR_PRESENT_POSITION = 132
LEN_PRESENT_POSITION = 4
DEFAULT_POS_SCALE = 360.0 / 4096.0  # 12-bit → 360°

dxl_present_position = group_bulk_read.getData(dxl_id, ADDR_PRESENT_POSITION, LEN_PRESENT_POSITION)
servo_joint_angles[dxl_id] = dxl_present_position * DEFAULT_POS_SCALE
```

```latex
\theta_{servo,deg} = \text{code}_{12bit} \times \frac{360°}{4096}
```

- 5 个 Dynamixel 舵机 (ID 0–4), 3Mbaud, Protocol 2.0。
- GroupBulkRead 一次性读 5 个舵机。
- 舵机位置代表 **MCP bend** (掌指关节弯曲), 对应力反馈的输入。

#### 4.1.3 URDF 关节变换 ✅

`udp_receiver.py` — 关节角度到 URDF 弧度的变换:

```python
_JOINT_TO_URDF_TRANSFORMS = (
    (0, 90.0, -1),   # thumb_dip
    (1, 270.0, 1),   # thumb_pip
    (2, 180.0, 1),   # thumb_mcp_s
    ...
)
_SERVO_TO_URDF_TRANSFORMS = (
    (0, 182.2, -1),  # thumb_mcp_b
    (1, 165.2, -1),  # index_mcp_b
    ...
)

def _to_urdf_radians(raw_angle_deg, zero_offset_deg, sign):
    return math.radians(sign * (raw_angle_deg - zero_offset_deg))
```

```latex
\theta_{URDF,rad} = \text{sign} \times (\theta_{raw,deg} - \theta_{zero,deg}) \times \frac{\pi}{180}
```

- 每关节独立 **零点偏移** (zero_offset_deg) 和**方向符号** (sign)。
- 硬件零点 (机械安装位置) 通过这些常量补偿。
- 16 个编码器关节 + 5 个舵机关节 = **21 个 URDF 关节**。

#### 4.1.4 FK → IK 引擎 ✅

`fk_ik_core.py` — **两阶段架构**:

```text
Stage 1 (FK): DOGlove-v3.xml (21 DOF 手套模型)
  → UDP joints (21 DOF) → mj_step → 5 个 fingertip site 位置

Stage 2 (IK): 目标机械手模型 (Inspire/Leap/Shadow/Allegro)
  → mink.FrameTask (5 个 fingertip, position_cost=1.0, orientation_cost=0.0)
  → mink.PostureTask (cost=1e-2, 正则化)
  → QP 求解 → 目标手关节角
```

**关键参数** (`ModelSpec`):
- `offset`: FK 到 IK 的全局平移 (如 leaphand: (-0.055, 0.15, -0.24))。
- `scale`: 统一缩放因子 (如 1.5), README 强调先调 scale 再调 offset。
- `tip_position_adjustments`: 每指尖残差平移。

```latex
\vec{p}_{target} = \text{scale} \cdot (\vec{p}_{FK} + \vec{offset}) + \vec{adjust}_{tip}
```

#### 4.1.5 LRA 力反馈控制 ✅

`glove_mcu.py` `_listen_lra()`:

```python
received_kp = struct.unpack("f" * 4, data)  # 4 个 kp 值 (UDP)
for i, kp in enumerate(received_kp):
    if 10 < kp < threshold:      # 有接触力
        self.lra_control(i, 56, int(1000 / self.write_fps))  # wave=56
    else:                         # 无接触
        self.lra_control(i, 222, 100)  # wave=222
```

- **阈值控制** (非 PID): kp 在 (10, threshold) 区间触发振动波形 56, 否则波形 222 (空振)。
- LRA (Linear Resonant Actuator) 通过波形号控制振动模式。
- 写帧格式: `[0x55, 0xAA, channel, wave, duration_h, duration_l, checksum]`。

### 4.2 数据结构/通信协议

#### 4.2.1 MCU UART 帧 ✅

`glove_mcu.py`:

```text
Header: AA 55 00 00 (4 bytes)
Data:   <I 16i I> = 4 + 16×4 + 4 = 72 bytes
Total:  76 bytes (block_size)
```

| 偏移 | 类型 | 含义 |
|------|------|------|
| 0–3 | uint8×4 | Header (AA 55 00 00) |
| 4–7 | uint32 (LE) | reference_voltage (µV) |
| 8–71 | int32×16 (LE) | 16 个编码器 ADC 值 |
| 72–75 | uint32 (LE) | CRC: 高 16 位 = checksum, 低 16 位 = timestamp |

**CRC 校验** ✅:
```python
checksum = (crc >> 16) & 0xFFFF
timestamp = crc & 0xFFFF
expected_checksum = sum(data[:-1]) & 0xFFFF
```

```latex
\text{checksum} = \left(\sum_{i=0}^{N-1} \text{word}_i\right) \mod 2^{16}
```

#### 4.2.2 LRA 控制帧 ✅

`glove_mcu.py` `lra_control()`:

```text
55 AA channel wave dur_h dur_l checksum
```

```latex
\text{checksum} = (\text{channel} + \text{wave} + \text{dur}_h + \text{dur}_l) \mod 256
```

注意: **入站帧头 AA 55**, **出站帧头 55 AA** — 方向相反, 防混。

#### 4.2.3 UDP 协议 ✅

`udp_receiver.py`:

| 端口 | 内容 | 格式 |
|------|------|------|
| 5009 | joint data | 16×float32 LE (64 bytes) |
| 5010 | servo data | 5×float32 LE (20 bytes) |
| 5012 | LRA kp | 4×float32 LE (16 bytes) |

所有 localhost (127.0.0.1), UDP。

#### 4.2.4 DOGlove FK 关节名 ✅

`fk.py` — 21 个 MuJoCo 关节:

```text
thumb:  bend_1, bend_2, split, mcp, bend_3 (5)
index:  bend_1, bend_2, split, bend_3    (4)
middle: bend_1, bend_2, split, bend_3     (4)
ring:   bend_1, bend_2, split, bend_3     (4)
pinky:  bend_1, bend_2, split, bend_3     (4)
Total: 21 DOF
```

### 4.3 硬件拓扑

```text
                    DOGlove 系统拓扑
  ┌──────────────────────────────────────────────────────┐
  │                   手套侧                              │
  │  ┌─ 旋转磁编码器 ×16 ──┐    ┌─ Dynamixel ×5 ──┐      │
  │  │  (每指 3 bend+1split) │    │  (每指 1 MCP bend)│     │
  │  │  24-bit ADC           │    │  12-bit, 3Mbaud  │     │
  │  │  SPI → STM32          │    │  RS485 → USB     │     │
  │  └──────┬───────────────┘    └──────┬──────────┘      │
  │         │ UART 921600               │ USB Serial      │
  │         ▼                           ▼                 │
  │  /dev/tty.wchusbserial*      /dev/tty.usbserial-*     │
  └─────────┬───────────────────────────┬─────────────────┘
            │                           │
  ┌─────────▼───────────┐  ┌───────────▼──────────────┐
  │ glove_mcu.py         │  │ servo.py                 │
  │ UART → UDP:5009      │  │ Dynamixel SDK → UDP:5010 │
  │ + LRA control (5012) │  │                          │
  └─────────┬───────────┘  └───────────┬──────────────┘
            │                           │
            └───────────┬───────────────┘
                        ▼
              ┌─ udp_receiver.py ─────┐
              │ UDP:5009 (16 floats)  │
              │ UDP:5010 (5 floats)   │
              │ → URDF joint transform │
              │ → 21 URDF radians     │
              └──────────┬────────────┘
                         ▼
              ┌─ fk_ik_core.py ──────────┐
              │ Stage 1: DOGlove-v3 FK   │
              │ → 5 fingertip positions  │
              │ Stage 2: mink IK         │
              │ → target hand joints     │
              │ (Inspire/Leap/Shadow/    │
              │  Allegro)                │
              └──────────┬──────────────┘
                         ▼
              机器人控制 (Franka 等)

  LRA 反馈环:
  机器人接触力 → kp (UDP:5012) → glove_mcu.py
    → lra_control(channel, wave, duration) → LRA 振动
```

**关键约束**:
- MCU UART: **921600 baud**, 二进制帧, 76 字节/帧。
- Dynamixel: **3Mbaud**, Protocol 2.0, GroupBulkRead。
- LRA: 5 通道, 波形 0–255, 时长 0–65535。
- 电源: 编码器 + STM32 + Dynamixel + LRA — 待人工核对 (PCBA 文件在 OSHWHub)。

### 4.4 关键 API 签名

```python
# glove_mcu.py
class UARTReader(UARTReaderBase):
    def process_block(self, block: bytes) -> None:
        # 输入: 76 字节帧 (header + data + CRC)
        # 输出: joint_angles → UDP:5009
        # 前置: buffer 中已找到 header

    def lra_control(self, channel: int, wave: int, duration: int) -> None:
        # channel: 0-4 (5 通道)
        # wave: 0-255 (LRA 波形)
        # duration: 0-65535
        # 前置: serial_port 已开

# servo.py
class ServoReader(UARTReaderBase):
    def _loop_once(self) -> None:
        # GroupBulkRead 5 个 Dynamixel
        # 输出: servo_joint_angles[5] → UDP:5010

# udp_receiver.py
class UDPReceiver:
    def get_most_recent_joints(self) -> Optional[tuple]:
        # 返回 21 个 URDF 弧度 (16 编码器 + 5 舵机变换后)
        # 前置: joint + servo 数据都已到达

# fk_ik_core.py
class FkIkEngine:
    def step(self, joints: Optional[Sequence[float]]) -> None:
        # 输入: 21 URDF 关节弧度
        # FK → fingertip positions → IK → 目标手关节
```

### 4.5 工程陷阱

1. **16 通道 → 5 指 21 关节映射不明确** 🔬: `glove_mcu.py` 读 16 个编码器值, `fk.py` 用 21 个关节名。_JOINT_TO_URDF_TRANSFORMS 只映射 16 个 (索引 0–15), 5 个舵机另算。但 16 = 3×5 + 1? 还是 4×4? 待人工核对 (STM32 固件在另一 repo: `doublehan07/DOGlove_Firmware`)。
2. **参考电压为零时跳过** ✅: `joint_angles = [val / reference_voltage * 360 for val in voltages if reference_voltage != 0]` — `reference_voltage == 0` 时列表为空, 不报错但下游收到空列表。
3. **UART buffer 同步** ✅: `process_buffer()` 找 header, 校验帧间距 == block_size。丢失/损坏帧直接跳过, 不重传。
4. **LRA 阈值控制非连续** 🟡: `10 < kp < threshold` 二值判定, 无渐变。波形 56 vs 222 硬切换, 可能产生振动手感不连续。
5. **UDP 无序号/时间戳** 🟡: joint/servo UDP 包无序号, receiver 只保留最新。丢包无检测。录制工具 (`udp_record.py`) 加了 `t_ns` 但运行时不用。
6. **FK→IK scale 调参敏感** ✅: README 明确警告 "Tune scale first", 不同机械手 scale 不同 (Leap/Allegro=1.5, Shadow=1.0)。
7. **tracker.py quaternion 顺序** ✅: `get_pose_quaternion()` 返回 `[x,y,z,qx,qy,qz,qw]`, 非 `[qw,qx,qy,qz]`。tracker.md 有警告。
8. **STM32 固件不在本 repo** ✅: 固件在 https://github.com/doublehan07/DOGlove_Firmware, 本 repo 只有 Python 上层。

---

## 5. 三源交叉对比表

| 维度 | LucidGloves | OpenGloves-Driver | DOGlove |
|------|-------------|-------------------|---------|
| **传感方式** | 旋转霍尔/电位计 (弦驱动) ✅ | 无传感 (驱动层) | 旋转磁编码器 (24-bit) ✅ |
| **DoF 数** | 5 curl (+5 splay 可选) = 5–10 ✅ | 继承固件 (5–10) | 16 编码器 + 5 舵机 = 21 ✅ |
| **关节粒度** | 每指 1 curl (整指) 🟡 | 结构支持 4/指, 实际 1/指 🟡 | 每指 3 bend + 1 split ✅ |
| **力反馈** | 舵机 (5× PWM) ✅ | 力反馈 + 触觉振动 (协议层) ✅ | Dynamixel 舵机 + LRA ✅ |
| **成本 (单手)** | ~$20–50 (ESP32+电位计+舵机) 待人工核对 | N/A (软件) | 待人工核对 (STM32+编码器+Dynamixel, 论文称 low-cost) |
| **精度** | 12-bit ADC, 弦驱动有间隙 🟡 | — | 24-bit 磁编码器, 直驱 ✅ |
| **通信协议** | Alpha (文本, \n 分隔) ✅ | Alpha (PC 端解析) + Named Pipe ✅ | 二进制 UART 76B 帧 + UDP ✅ |
| **传输方式** | USB Serial / BT SPP / WiFi ✅ | Win32 Serial / BT / Named Pipe ✅ | USB UART 921600 + UDP localhost ✅ |
| **平台** | ESP32 / AVR ✅ | Windows only ✅ (Linux stub) | 跨平台 Python ✅ |
| **生态** | SteamVR (VR) ✅ | SteamVR (VR) ✅ | MuJoCo + 机械手 IK ✅ |
| **目标场景** | VR 手部追踪 | VR 驱动桥接 | 灵巧操作遥操作 |
| **固件开源** | ✅ Arduino .ino | N/A | 🔬 STM32 (另一 repo) |
| **IK/FK** | 无 | 无 | ✅ mink (多机械手) |
| **对 EgoGlove 相关性** | **高**: 低成本 flex/霍尔 + ESP32 + Alpha 协议参考 | **中**: SteamVR 集成模式、设备发现、姿态校准 | **高**: 21 DOF 编码器、FK→IK 机械手映射、力反馈闭环 |

---

## 6. 对 EgoGlove 的启示

### 6.1 直接可借鉴 ✅

1. **LucidGloves Alpha 协议**: 文本编码, 调试友好, \n 分隔。EgoGlove Hand Token v2 用 TLV 二进制更紧凑, 但 Alpha 的字段标签设计 (A–E curl, (AB)–(EB) splay, (AAA)–(EAD) joint) 可作为**人类可读调试模式**的参考。

2. **DOGlove FK→IK 两阶段**: 先 FK 到手套空间 (21 DOF), 再 IK 到目标机械手。EgoGlove 的 **MANO Layer → Robot Action Layer** 双表示层与此同构:
   - MANO Layer ≈ DOGlove FK (手套→人体手)
   - Robot Action Layer ≈ DOGlove IK (人体手→机器人手)
   - `ModelSpec` (offset/scale/tip_adjustments) 的调参方法论可直接复用。

3. **DOGlove URDF 变换表** `(source_idx, zero_offset, sign)`: 每关节独立零点+方向。EgoGlove 的 canonical-20 旋转关节需要类似的**机械安装校准表**。

4. **LucidGloves 动态 min/max 校准 + Travel 限制**: 适用于 flex 传感器 (Lite), 但对旋转编码器 (Pro) 不需要 — 编码器是绝对角度。

5. **DOGlove 二进制帧 + CRC**: 比 Alpha 文本协议带宽效率高 ~10×。76 字节/帧 vs Alpha ~60 字符/帧, 但编码器数据量更大。

### 6.2 需改造 🟡

1. **LucidGloves 单 curl/指 → EgoGlove canonical-20**: LucidGloves 每指一个弯曲值, EgoGlove Hand Token v2 需要 20 个旋转关节。DOGlove 的 3 bend + 1 split/指 更接近, 但仍需 MANO 映射。

2. **Alpha 编码 flexion[4] 预留 → 实际多关节**: opengloves-driver 结构已支持 `array<array<float,4>,5>`, 但固件不填。EgoGlove 若走 Alpha 兼容路线, 可利用 `(AAA)`–`(EAD)` 多关节键。

3. **LucidGloves 舵机力反馈 → EgoGlove Pro**: LucidGloves 用 PWM 舵机 (廉价但粗糙), DOGlove 用 Dynamixel (精确但贵)。EgoGlove Pro 力反馈方案待定 — 🌌 长期方向。

### 6.3 风险/约束 🔬

1. **LucidGloves 弦驱动机械精度**: 鲍登线有弹性、蠕变、间隙。6 个月后弦可能松弛, 校准漂移。EgoGlove Lite 用 flex 传感器无此问题, 但 flex 本身有迟滞。

2. **DOGlove 16→21 映射待核实**: 编码器物理布局与 URDF 关节的对应关系需对照 STM32 固件。本 repo 未包含固件源码。

3. **opengloves-driver Windows-only**: EgoGlove relay 用 Python FastAPI 跨平台, 不能直接复用 Win32 串口/BT 代码, 但协议层 (Alpha 解析) 可移植。

4. **DOGlove 论文 "low-cost" 定义待核对**: 含 STM32 + 16 编码器 + 5 Dynamixel + LRA, 实际 BOM 待人工核对。可能与 EgoGlove Lite <¥500 目标不在同一量级。

### 6.4 关键文件复用清单

| EgoGlove 需求 | 参考源 | 文件 |
|---------------|--------|------|
| Lite 固件框架 | LucidGloves | `Main.cpp`, `InputManager.cpp` |
| 串口/BT 通信 | LucidGloves | `SerialCommunication.cpp`, `BTSerialCommunication.cpp` |
| flex 校准算法 | LucidGloves | `InputManager.cpp:111-145` |
| Alpha 协议参考 | LucidGloves + OpenGloves | `AlphaEncoding.cpp`, `alpha_encoding_service.cpp` |
| SteamVR 姿态校准 | OpenGloves | `pose_calibration.cpp` |
| FK→IK 双表示层 | DOGlove | `fk_ik_core.py` |
| 关节零点变换 | DOGlove | `udp_receiver.py` _JOINT_TO_URDF_TRANSFORMS |
| UDP 数据流架构 | DOGlove | `glove_mcu.py`, `udp_receiver.py` |
| 数据录制/回放 | DOGlove | `tools/udp_record.py`, `tools/udp_replay.py` |

---

## 附录: 真实性声明

- LucidGloves 与 OpenGloves-Driver 代码已逐文件阅读, 标注 ✅ 的项有源码佐证。
- DOGlove **STM32 固件不在本 repo** (https://github.com/doublehan07/DOGlove_Firmware), 所有关于编码器物理布局、ADC 采样率、电源管理的描述标注为 🔬 或待人工核对。
- LucidGloves **Wiki** (https://github.com/LucidVR/lucidgloves/wiki) 未直接访问, 机械装配细节来自 README + STL 文件名推断, 标注待人工核对。
- DOGlove 论文 (arXiv:2502.07730) 未直接阅读, "low-cost" 定义和性能指标标注待人工核对。

---

*End of research distillation.*
