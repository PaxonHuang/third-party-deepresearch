# third-party-deepresearch — EchoGlove 研究资料集

跨会话共享的研究资料（非产品仓；产品代码在 `../EgoGlove` 与 `../EchoGlove-SLR-MOCAP-Beta`）。

## 结构

| 目录 | 内容 |
|------|------|
| `deepresearch/` | **EchoGlove 科研级原理文档**（2026-08-14 自 `EchoGlove-SLR-MOCAP-Beta/docs/deepresearch/` 迁入）|
| `paper/` | 研究论文笔记（`2509FSGlove.md`、`2512OSMO.md`）|
| `repo/` | 第三方开源仓库参考（**gitignored**，见下）|

## deepresearch/ — EchoGlove 原理文档（9 主题 + index）

`01_signal_chain` 信号链 · `02_imu_fusion_madgwick` IMU+Madgwick（含 2026-08-11 梯度符号修正决策）· `03_hand_token_protocol` Hand Token v1 · `04_l1_edge_cnn` · `05_l2_gated_bi_cross_attn` · `06_l3_stgcn_vision` · `07_training_quant` · `08_nlp_tts` · `09_fk_mano_dual_rep` · `index.md`（全景 + 真实性总表）

> 每篇含 LaTeX 核心公式 + 带张量维度的科研级 ASCII 网络图 + 四级真实性标注（✅/🟡/🔬/🌌）。依据 `EgoGlove/docs/superpowers/specs/2026-08-10-egoglove-aligned-production-design.md` 与当前代码。

## repo/ — 第三方源码参考（gitignored，从上游重新获取）

`.gitignore` 排除 `repo/`（含大体积 zip 与第三方提取源码，避免重复版本化）。参考清单（zip 文件名）：

- `DOGlove-main.zip`（提取：`DOGlove-main/`）
- `dex-retargeting-main.zip`
- `dexbotic-main.zip`
- `osmo_tactile_glove-main.zip`
- `wuji-sdk-main.zip`
- `LucidOpenGlove/`

如需将某份源码纳入版本管理，取消 `.gitignore` 对应行并提交（注意体积）。

## 维护纪律

- 提交身份 `PaxonHuang <quenchkidney@outlook.com>`；`type(scope): description`。
- 本仓为资料集，不承载产品代码；产品工作见两个产品仓。
