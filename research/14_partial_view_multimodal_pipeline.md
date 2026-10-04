# 14：局部腕视角与多模态事件学习研究协议

🔬 本文是通用研究方法和原始文献评述，不代表任何产品已训练、部署或达到论文性能。研究重点先设为接触事件与动作区间，再分别扩展局部跟踪、物体理解和关节姿态。

## 1. 不完整手结构与视觉任务选择

近腕局部摄像可能看到强透视、截断手指、遮挡和模糊。它们是需要对照实验的假设，零检测不能直接归因为“ROI 太少”。MediaPipe 掌检测后裁剪、再预测关键点；裁剪放大不能恢复画外关节。[官方任务说明](https://ai.google.dev/edge/mediapipe/solutions/vision/hand_landmarker)

🟡 保持模型、版本、SHA-256、阈值、IMAGE/VIDEO 模式、旋转、裁剪、输入色彩、焦距、曝光和分辨率有固定 manifest。用同一相机拍完整裸手作正对照，无手背景作负对照；正对照失败先查软件和光学。之后依次只改变俯仰角、可逆近端位移、手套外观，最后恢复基线；记录实际角度/位移和误差。安装变化需检查外参适用性。

盲标 palm/joint visible、occluded、out_of_frame，以及 blur/exposure。按条件和可见性报告召回、误检、有效跟踪长度和重捕获时间。检测覆盖不是几何准确度，内部 forward/backward 一致性也不是独立 GT。

局部追踪可用 Shi–Tomasi + LK：

$$\hat d=\arg\min_d\sum_{x\in W}(\nabla I(x)^T d+I_t(x))^2,$$

$$e_{FB}=\|x_t-\hat x_t^{back}\|_2.$$

追踪失效显式结束片段，重初始化单独记录；用独立盲标检查点评价 pixel error、有效覆盖和 fragmentation。[OpenCV 官方说明](https://docs.opencv.org/4.x/d4/dee/tutorial_optical_flow.html)

## 2. 事件任务与真实标签

🟡 接触标签与动作标签分开；提示语只是意图，不是实际执行边界。接触包括 onset/hold/release、digit 与可见性；动作包括 idle、open_close、wrist_rotation、press 及 transition/ambiguous 状态。组合动作保留多标签；首版单类基线排除 composite 窗口并单列 stress set，报告排除数量和全程覆盖，不能把并行动作硬分为单一 softmax 类。

外部普通相机可标可见接触和动作，但不提供力幅值或毫米级 3D 真值。保留原视频、帧时间与曝光可用性；共同可见光学事件可用于相机间 $t_B=a t_A+b$ 关联，记录 residual、valid segment 和 uncertainty。没有独立共同时刻证据时，不宣称触觉/惯性对齐；host arrival 未知源延迟保持未知。

边界用最早/最晚可能时刻表示，包含帧间隔、曝光、人工选择和映射误差。被遮挡或时间不可辨的事件标 ambiguous；不使用待评算法输出自举测试真值。

## 3. 可执行小数据流水线

🟡 至少三次独立脱下再穿戴的 pilot session，每次独立校准；每动作至少十次、随机次序，覆盖接触静止、无接触运动、无接触静止、快动作、遮挡与卸载。单被试结果只称工程 pilot，跨人泛化另采被试。

1. **数据与标签**：raw 不变；标签保留 source、reviewer、protocol revision、visibility、时间不确定性、裁决和 raw 引用。默认单人审查明确 single_review；可获得第二人时做独立复核，均先不看预测。
2. **清洗与版本**：每个 raw/label/split/feature/model manifest 带 hash 与父版本；更正另存新版本。生成 error/gap/saturation/occlusion/mount/calibration mask，不静默删除或把 unknown 填零。
3. **切分**：先按 subject/session/donning 切分，再窗口化。三 session pilot 分别 train/validation/test；相邻和重叠窗口不能跨 split。窗口不跨 gap 或校准边界。测试不足须报告，不重新混入训练。
4. **特征**：触觉 bias-subtracted activity、活动点数、传感器轴空间分布；gyro/acc 的 mean/std/RMS；局部 flow 和有效点数。带 coverage、age、max gap、modality mask。没有外参不叫世界力，没有 sensor-to-bone 不叫关节角。
5. **学习**：先冻结规则状态机，再正则逻辑回归；train 拟合标准化，validation 调参，test 冻结。只有独立标签/基线/误差分析完成后才比较小型 causal TCN。
6. **部署**：训练离线，采集与推理分离、有界队列、age/timeout 输出 unknown。保存模型与配置 hash，先 replay 一致性后在合格采集环境运行推理；recorder/inference/preview loss 分别记录。

活动特征可写为 $A_s(t)=\sum_j\|f_{sj}(t)-\bar f_{sj}\|_2$，它不是合力。规则基线使用卸载 median/MAD 门限、迟滞、dwell 和显式 gap/censoring；门限取自 train 或预定义的部署前卸载段，不取测试动作标签。逻辑回归 $p(c|z)=\operatorname{softmax}(Wz+b)$，最小化交叉熵加 $L_2$。

## 4. 评价与贡献审查

🟡 报 contact event precision/recall、动作 macro F1/混淆矩阵、segment IoU、unknown 比例及每 session 结果。事件用冻结的一对一重叠匹配；有有效时域才算 onset/end error，并展示 label uncertainty。不确定事件不能自动算负例。对 tactile-only、IMU-only、vision-only、组合模态、quality gating 开/关做同 split 消融；披露各模态可用率，不能只挑“全有效”帧。

按 session/subject 重采样区间，而非把相邻帧当独立样本；三个 session 不足以证明稳定统计泛化。部署另报 p50/p95 延迟、age、CPU/RAM、温度和可测功耗、推理 skip。3D pose/关节误差和力 MAE 需要各自独立且可追溯 GT；普通参考视频不能支持这些声明。

🔬 可能贡献应限定为经验证的观测可追溯性、时间/质量/可观测性门控、以及部分视觉—惯性—直接触觉的互补性。标准 LK、MAD、逻辑回归本身不是新算法；腕摄像与 pose/pressure 联合也已有工作。系统工程完成、性能增益和新颖性要分别举证。

## 5. 原始来源与许可状态

下表固定论文版本，仅核对摘要/官方文档，未作代码复现。代码、论文、模型、数据的许可分别核对；unknown 不等于禁止引用，也不等于允许复制/分发资产。

| 原始来源 | 版本与有关结论 | 许可与可复用状态 |
|---|---|---|
| [MediaPipe Hands](https://arxiv.org/abs/2006.10214v1) | v1；palm detector + hand landmark pipeline | 官方文档文字 CC-BY-4.0、示例代码 Apache-2.0；论文再分发/具体源 commit/下载模型许可本轮未核验 |
| [Recognition from Hand Cameras](https://arxiv.org/abs/1512.01881v3) | v3；腕摄像可研究手活动、手状态和物体类别，不必等完整关键点 | 论文再分发、代码、模型和数据许可 unknown |
| [WildHands](https://arxiv.org/abs/2312.06583v2) | v2；近距离透视、遮挡、模糊和缺3D标注是域挑战；相机信息值得研究 | 具体源 commit、模型、数据及论文再分发许可 unknown；头戴结果不能直接外推腕戴 |
| [WristPP](https://arxiv.org/abs/2603.00606v1) | v1；wide-FOV腕摄像联合3D pose/per-vertex pressure；ViT joint-aligned tokens、Hand-VQVAE、extrinsics-conditioned branch | 代码/模型/数据/论文再分发许可 unknown；只核对摘要，不引用性能为本地预期 |

WristPP 必须用于 novelty 排重：不能宣称“首个腕摄像联合姿态压力”。后续复用任何实现前，另记录源 commit、完整资产许可证与复现实验条件；论文指标是作者实验结论，不是本地验收。
