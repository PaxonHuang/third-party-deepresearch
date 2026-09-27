# ADS1115 — historical 16-bit I²C ADC candidate evaluation (inactive)

> **2026-09-27 applicability:** **Historical candidate, inactive.** The dated evaluation below is retained as a research record. It is not a current procurement, wiring or firmware instruction and does not authorize adoption of an ADC or a private project topology. See [transport and timing lessons](13_edge_acquisition_transport_timing.md).


> **提炼日期**: 2026-09-14 · **提炼人**: PaxonHuang <quenchkidney@outlook.com>
> **上游**: TI SBAS444E datasheet（ADS1113/1114/1115，MAY 2009 – REVISED 2024-12），官方 PDF 已入库：`/home/EchoGloveHugeProjects/datasheet/TI_ADS1115_SBAS448_datasheet.pdf`（commit `199896f`）。
> **历史角色**: 2026-09-14 的 prototype/candidate 评估；当前非活动，不作采购或接线指令。
> **架构边界（不得推翻）**: EVT0 选定架构是 **external 16-channel ADC + 4:1 analog row switch**（`demo2a-evt0-electrical-decision.md` §4）。ADS1115 是 **4 通道 I²C ADC**，与 16 通道 SPI ADC 能力**不等价**；本篇评估其作为 EVT0 实验垫脚石的适配性，不作为最终架构答案。

## 0. 结论速览（.datasheet-derived，全部待实测复核）

🔬 **ADS1115 适合 EVT0 早期接触/位置证据链，不适合作为最终 16-channel tactile backend**：

| 维度 | ADS1115 (datasheet-derived) | EVT0 选定架构需求 | 判定 |
|---|---|---|---|
| 通道数 | 4 单端 / 2 差分 | 16 直入 + 4:1 行开关 | ❌ 差 4 倍，必须保留 future footprint |
| 接口 | I²C（占用 acquisition bus） | SPI（独立于 sensor bus） | ⚠️ I²C 带宽挤占 IMU 总线预算 |
| 采样率 | 8–860 SPS | 单模块 60–100 帧/s × 16 cell ≈ ≥1.6 kSPS | ❌ 860 SPS 上限不足（见 §2 计算） |
| 分辨率 | 16 bit ΔΣ，PGA ±0.256…±6.144 V | 16 bit 类 | ✅ 满足分辨率类 |
| 典型用途 | 低速多点温度/电压监测、电池计量 | 高帧率触觉矩阵扫描 | ⚠️ 仅覆盖"慢帧接触定位"实验 |

**使用方式**：EVT0 前期用 1 片 ADS1115 接 1 个 tactile 模块（4 列输入直入 AIN0–3，行开关 GPIO 扫描），先建立 *接触定位/状态* 的最小证据链与驱动路径；**Schematic 同时保留 16-channel SPI ADC + 第二个 4:1 开关的 DNP footprint/connector**（电气决策文档 §4 "Selected + Conservative expansion" 行），ADS1115 不删除该边界。ADS1115 footprint 标注 `ADC_I2C_CANDIDATE`（可 DNP），最终 ADC 决策回到采购/噪声评审。

## 1. 核心规格与公式（datasheet-derived）

🔬 **转换模型**（SBAS448 §9.5.3–9.5.5，待人工核对页码）：

$$\text{Code} = \frac{V_{AIN}}{FSR_{PGA}} \times 32767, \quad FSR_{PGA} \in \{6.144, 4.096, 2.048, 1.024, 0.512, 0.256\}\,\text{V}$$

- **LSB**（±2.048 V 满量程，最常用）：$LSB = 2.048/32767 ≈ 62.5\,\mu V$。
- **PGA 供电约束**：PGA 满量程不得超过 VDD + 0.3 V；负输入不低于 GND − 0.3 V（无负轨能力，单电源系统 AIN 只能 0–FSR）。
- **数据速率**：DR 位 8/16/32/64/128/250/475/860 SPS；**860 SPS 对应 12 bit 有效分辨率**（带宽 3000 Hz class），16 bit 有效分辨率仅在 8–128 SPS 档（ENOB ≈ 15.3 bit @128 SPS class，datasheet 噪声表）。
- **建立时间**：ΔΣ 单次转换按所选 DR 周期完成；切换 MUX 后第一帧即有效，但**输入 RC + 开关建立必须另行预算**（datasheet 无外部 mux 建立保证——EVT0 需实测，见 §5）。
- **I²C 时序**：标准 100 kHz / 快速 400 kHz；4 地址引脚 → 地址 0x48–0x4B，**与 TCA9548A 下游共存无地址冲突**（LSM6DSV16X 0x6A/0x6B，ADS1115 0x48–0x4B，待人工核对板级 strap 分配）。
- **ALERT/RDY 引脚**：可配置转换就绪或比较器输出（OS watch）；EVT0 可用作 DRDY 式采样同步参考（GPIO 中断），但**不是**跨域同步语义——source clock domain 仍归 MCU tick。
- **电流**：active ≈ 150 µA class、shutdown ≈ 0.5 µA class（datasheet 电气表）；对 3V3_TACT 预算可忽略级，待实测。

## 2. 帧率可用性计算（datasheet-derived，EVT0 实验 target class）

EVT0 触觉矩阵帧率模型（bring-up matrix 已定义）：

$$f_{frame} = \frac{1}{N_{cell}\,(t_{convert} + t_{settle}) + t_{row/control}}$$

以 ADS1115 @860 SPS（$t_{convert} ≈ 1.16\,ms$，假设 $t_{settle} ≈ 0.2\,ms$ 需实测）：

- **单 4×4 模块、行扫描**：每帧 16 cell，但 ADS1115 每次只采 1 通道 → $f_{frame} ≈ 1/(16×1.36\,ms) ≈ 46\,frame/s$ —— 落在 "60–100 帧/s" 目标类**之下**。
- **ADS1115 4 通道直入 + 行开关**：每行 4 列并行采（AIOS 复用仍逐通道），等效每帧 4 次转换/行 → 结构上仍是顺序转换，无并行收益；ADS1115 **没有同步多通道同时采样能力**（无 simultaneous sampling）。
- **5 模块串行**：≈ 9–12 完整帧/s —— 低于 "12–20 帧/s" 目标类。

🔬 **判定**：ADS1115 860 SPS 档只能支撑 **约 30–45 完整帧/s 的单模块接触定位实验**（位置/状态语义，电气决策 §3 EVT0 claims 范围内）；达不到选定 16-channel ADC 方案的帧率目标。这**不是 PASS**，是 candidate 边界内的实验能力上限。

## 3. 硬件拓扑（EVT0 candidate 接入，transport alias）

```
  3V3_TACT ──保护/去耦──┬────────────────────────────┐
                        │                            │
              ┌─────────┴────────┐         4:1 SW ────┤ row excitation
              │ ESP32-S3         │          (GPIO sel) │ (module rows)
              │ I2C1 (upstream)  │              │      │
              │   │              │        ┌─────┴──────┴───┐
              │ TCA9548A-B ──────┼────────┤ tactile module │
              │   ch: tact 4x4   │        │ 4 cols → AIN0..3│
              └──┬───────────────┘        └─────┬──────────┘
                 │ SDA/SCL (downstream alias)   │
           ┌─────┴──────┐                        │
           │ ADS1115    │  addr 0x4A (strap 待人工核对)
           │ ADC_I2C_   │  ALERT/RDY → ESP32 GPIO (DRDY 参考)
           │ CANDIDATE  │  VDD=3V3_TACT, GND=AGND
           └────────────┘  0.1µF decoupling + 10µF bulk
```

- I²C 挂在 **TCA9548A-B 的 tactile 通道**下（与 IMU 隔离，select-once + per-mux lock 语义不变）——也可评估直接挂 I2C1 上游以省一次 mux 切换；**默认画在 mux 下游**，实测带宽后再定（transport alias，不影响语义）。
- AIN0–3 = 模块 4 列模拟输出；行由 GPIO 控制的 4:1 低漏开关逐行激励。**当前 EVT0 populated channels = 1 模块 4 列 × 4 行**；future expansion = 第二个开关 DNP + 16-channel SPI ADC footprint 保留（ADS1115 位置可 DNP）。
- 每路 AIN 加 series R（≥1 kΩ）+ 至 AGND 的 RC（建议 1–10 nF 起步，settle 实测定值）作输入保护/抗混叠；模块输出阻抗未知 → **待实测 settle 时间**。
- I²C 上拉只在 3V3_TACT 域一级（mux 下游 per-rail 上拉规则沿用工程包 §2）。

## 4. 驱动接口契约（实现侧，不抄实现体）

ESP-IDF/固件侧需要的最小契约（🟡 工程可实现）：

```c
// addr: 0x48..0x4B (ADDR strap); 配置寄存器 2×16bit write, 转换结果 16bit read
// key ops: set_mux(ain_x | ain_y | diff), set_pga(gain), set_dr(sps), start_single(),
//          poll_os(), read_conv() -> int16 code
// 语义约束: 返回 code + (sample_event_time, host_receive_time, dr_mode, pga, ain_alias)
//          —— code ≠ calibrated pressure; 标定前仅 raw 电平 + 接触状态类
```

- 帧语义务必带 **source event time（MCU tick）与 host receive time 分离**——ADS1115 无时间戳，事件时间由固件在 OS 置位时打戳；禁止用 I²C 到达时间冒充采样时间。
- 缺模块语义：mux 通道扫描 NACK / ALERT 无响应 → 显式 `absent/disabled/faulted` 状态，**不得零值填充**（bring-up matrix 规则）。

## 5. 工程踩坑清单（datasheet-derived + 通用 I²C 经验，均待实测）

1. **860 SPS ≠ 16 bit**：高速档 ENOB 掉到 ~12 bit。若 EVT0 需要 16 bit 有效分辨率，帧率上限直接掉到 128 SPS（≈ 7–8 完整帧/s 单模块）。**先定接触实验需要几位，再定 DR 档**。
2. **单次转换启动位（OS）必须在写配置时置 1**；连续模式功耗高且无法按帧触发——触觉扫描建议 single-shot + DRDY 中断。
3. **MUX 切换后首帧未定 settling**：datasheet 只保证内部 ΔΣ 建立；外部行开关 + 模块 RC 的建立时间必须实测（bring-up matrix 已列 `per-cell settle` 项）。首版固件在行切换后插入可配置 dummy delay 并记录为测量参数。
4. **PGA 满量程 < VDD 约束**：3V3 供电时不要选 ±4.096 V 以上 PGA 期待满摆幅输入（6.144 V 量程在 3V3 系统下永远达不到，等效降分辨率）。
5. **AIN 负压无能力**：单电源、无负轨；行激励方向/极性必须为正或 0，模块输出若有负摆需先做偏置（待人工核对模块输出特性）。
6. **I²C 时钟拉伸**：ADS1115 无 clock stretching 需求（转换期间只 NACK 或延迟 ACK 视模式），但 mux 下游挂载时注意 TCA9548A 本身的通道保持行为——select-once 语义下不要在转换中切通道。
7. **ALERT 引脚开漏**：需要上拉到读取域的电源；若挂在 3V3_TACT 而 ESP32 GPIO 在 3V3_DIG，注意跨域电平——同 3V3 系列可直接，但 test point 要分别标注。

## 6. 与产品仓语义的交叉核对

- 频率/帧率模型与 `EgoGlove/docs/hardware/demo2a-evt0-bringup-matrix.md`（§Tactile scan matrix）一致；f_frame 公式同源。
- "absent/disabled/faulted 不零值填充" 与 bring-up matrix 末行一致。
- 本篇不改 Hand Token v2 / canonical-20 / Demo1Frame 任何语义；ADS1115 的 code/时间戳仅进入 observation/provenance 层，作为 tactile backend adapter 输入，**不进 Hand Token**。
- 16-channel ADC + 4:1 开关架构边界见 `EgoGlove/docs/hardware/demo2a-evt0-electrical-decision.md` §4；本篇不推翻。

## 待人工核对项

- [x] TI SBAS444E PDF 已入库 `datasheet/TI_ADS1115_SBAS448_datasheet.pdf`（commit 199896f，MAY 2009 – REVISED 2024-12）
- [ ] EVT0 实物 ADS1115 模块形态（裸片/最小系统板）与 strap 现状
- [ ] tactile 模块（4×4）输出阻抗/极性 → settle 与偏置设计
- [ ] DR 档与 ENOB 的最终实验选择（接触定位实验所需分辨率）
