# 11. Multimodal Pose Architecture Lessons

> 状态：EgoGlove 内部 research distillation；不是第三方事实的替代品。
> 日期：2026-09-05
> 用途：为 Demo2A Multimodal Pose Architecture Spec v0.1 提供可追溯的长期工程经验。

## 1. 证据边界与目录职责

本摘要只蒸馏仓库已有 `third-party-deepresearch/paper/`、`repo/`、`research/` 与 EgoGlove V8 文档中的结论。`research/` 保存跨来源原理与限制；`advicebyAI/` 保存项目适配建议；`repo/` 是第三方源码/资料快照，不等于 EgoGlove 已接入。论文和第三方报告中的精度数字不是 EgoGlove 本地验证结果。

## 2. 对 Demo2 架构有直接影响的经验

### Calibration and observability

- **FSGlove / DiffHCal**：sensor-to-bone calibration 是一级问题；安装误差应作为显式 calibration object，而不是假设 PCB axis 等于 anatomical bone axis。可微 fitting 可联合估计安装误差、手型和运动，但论文报告的误差不能转写为 EgoGlove 性能承诺。
- **Kalibr / GTSAM / VINS-Mono**：当前本地 corpus 没有足够的一手条目，不能声称 EgoGlove 已采用这些工具。本 Spec 只冻结 camera–IMU temporal/spatial calibration、factor responsibility 和 sliding-window 可替换接口，不冻结具体库。
- **Project Aria / HumanEgo**：timestamp、calibration、container/health 和可追溯数据组织值得借鉴；这证明 HumanEgo 使用 Aria 数据管线，不证明 EgoGlove 已接入 Aria，也不赋予其模型/数据许可。

### Vision and multimodal measurements

- **MediaPipe / HaMeR / WiLoR / HumanEgo**：视觉输出应作为带 confidence、visibility、occlusion 和模型 provenance 的 measurement。wrist-mounted camera 对局部 articulation、遮挡修正和 contact/object context 有价值，但不能天然提供 global wrist 6DoF。
- **FlexiTac / 3D-ViTac**：触觉更适合表达 contact probability、pressure/force、centroid、normal 和 quality；不得把 tactile 直接升格为 joint-angle truth。当前 `repo/FlexiTac/` 主要是研究 gallery/资料，不是可直接复用的完整传感器实现。
- **hand-tracking-fusion-system / AnyHand / xio Fusion**：本地没有足够证据，不能写成已研究、已验证或已接入方案。

### Representation and downstream boundaries

- **AnyTeleop / Dex-Retargeting**：属于 human→robot projection/adapter；指尖/关键点等可作为跨 embodiment 的输入，但 retargeting 的机器人约束不应反向定义 Canonical Human Hand State。
- **UMI / RLDS / LeRobot**：EgoGlove V8 已把 Episode、Provenance 和 RLDS/LeRobot 作为 downstream projection boundary；本轮不实现 exporter、训练 pipeline 或 LeRobot 接入。
- **MANO**：MANO 可作为 derived/projected mesh or fitting representation；不能成为传感器真值或 canonical core。MANO 单独许可证/非商业约束必须与代码、checkpoint、数据许可分开登记。

## 3. 不可丢失的架构结论

1. 6-axis IMU 的 gravity 主要约束 roll/pitch；无磁情况下 yaw 长期不可绝对观测，只能是 relative/drifting，除非有外部/VIO/SLAM/reference source。
2. Sensor-to-bone extrinsic 是一级资产；`T_B^S`、`T_I^C` 必须来自 CalibrationArtifact，PCB/FPC axis 不能作为默认 bone axis。柔性材料变形导致 `T_B^S(t)` 时，固定外参假设失效。
3. Raw measurements、derived observations 和 final canonical pose 分层保存；host receive timestamp 不能冒充 sampling timestamp；clock domain 必须显式建模 offset 与 skew/drift。
4. Flex 是 bend constraint/measurement model；FlexiTac 是 contact/pressure/quality observation。
5. MANO 是 projection/model adapter，不是 canonical truth。
6. 长期可复用资产优先级是 **Timestamp + Extrinsic + Calibration + Quality + Provenance + Canonical representation**，而不是 IMU 数量本身。

## 4. 许可证与依赖风险

- FSGlove 本地论文/README 说明与源码许可完整性不能混为一谈；在缺少明确 LICENSE 时只研究思想，不复制源码/硬件设计。
- MANO 许可限制非商业科研/教育/艺术用途；HumanEgo 使用 PolyForm Noncommercial；模型 checkpoint、数据集、代码和 robot assets 必须分别核查。
- AnyDexRetarget/Dex-Retargeting 主代码许可相对清晰，但机器人 URDF/mesh/asset 可能是混合许可。
- FlexiTac gallery 的收录不代表条目代码或硬件可复用。

## 5. 对本项目的落点

以上经验已被蒸馏到 `docs/superpowers/specs/2026-09-05-demo2a-multimodal-pose-architecture-v0.1.md` 的 coordinate/timestamp/topology/calibration/factor/canonical-state/compatibility/gate 章节。本文不声称任何 Demo2A hardware、ground truth、solver 或 accuracy gate 已完成。

## 6. 主要本地来源

- `third-party-deepresearch/research/10_dex_mocap_teleop_tactile.md`
- `third-party-deepresearch/paper/2509FSGlove.md`
- `third-party-deepresearch/paper/2604.28156v1FlexiTac.md`
- `third-party-deepresearch/paper/2307.04577AnyTeleop.md`
- `third-party-deepresearch/paper/2605.24934v2_HumanEgoZero.md`
- `third-party-deepresearch/repo/fsglove/`
- `third-party-deepresearch/repo/FlexiTac/`
- `third-party-deepresearch/repo/HumanEgo/`
- `EgoGlove/docs/V8/00_HUMAN_MOTION_INFRASTRUCTURE.md`
- `EgoGlove/docs/V8/03_EPISODE_MODEL.md`
- `EgoGlove/docs/V8/04_PROVENANCE_MODEL.md`
