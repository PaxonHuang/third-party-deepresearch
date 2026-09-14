# Demo2A Upstream Source Manifest

> Date: 2026-09-07
> Purpose: reproducible, isolated evidence manifest for the Demo2A second architecture review.
> Isolation rule: source snapshots are under `third-party-deepresearch/repo/`, outside EgoGlove. They were not copied into EgoGlove and were not indexed by EgoGlove CodeBaseMemory or Graphify.

| Source | URL / pinned revision | License evidence | Research purpose | Reuse status | CBM status |
|---|---|---|---|---|---|
| FSGlove | `https://github.com/davidliyutong/fsglove/tree/a7988f051c35f5ee6157353ee4022d73aa1e4cc8` | no repository-wide LICENSE/COPYING/NOTICE found | inertial hand calibration/state/packet evidence | reference only; code reuse ungranted | not indexed |
| Kalibr | `https://github.com/ethz-asl/kalibr`, local `1f60227442d25e36365ef5f72cd80b9666d73467` | BSD 3-clause-style repository license | camera–IMU intrinsic/spatial/temporal calibration categories | reference only; no implementation adopted | not indexed |
| GTSAM | `https://github.com/borglab/gtsam`, local `8be77851e956f2087600950ac1b936e0d340440c` | BSD repository license | factor-graph optimization framing | reference only; no solver adopted | not indexed |
| manotorch | `https://github.com/lixiny/manotorch`, local `a2a70c591f91551078b7bb2af9b5d9f275b626e0` | Apache-2.0-labelled code; MANO model assets separately licensed | optional downstream parametric hand projection | code/model terms must be reviewed separately; not adopted | not indexed |
| hand-tracking-fusion-system | `https://github.com/alexandertianlin/hand-tracking-fusion-system/tree/165f40d0b7593884514eb80a68c58bb455637d43` | MIT code | bounded vision/IMU/tactile baseline and evaluation caution | reference only; pending metrics not performance evidence | not indexed |
| xio Fusion | `https://github.com/xioTechnologies/Fusion/tree/9325424011892abacc0ce42b8bb1a8ae20264b9b` / `v1.3.3` | MIT | AHRS bias/rejection/overrange/time/axis semantics | reference only; no library adopted | not indexed |
| MediaPipe | `https://github.com/google-ai-edge/mediapipe/tree/c17b2a83e8944d2811889a2a08d629c20bcb6ed8` | Apache-2.0 code; model/dataset/assets separate | camera measurement provenance and limitations | reference only; model terms unverified | not indexed |
| FlexiTac hardware | `https://github.com/FlexiTac/FlexiTac_Hardware_Repo/tree/e006b179e89b79be28dd7062c5dd75ed866e5e7a` | CC BY-NC 4.0 | tactile raw frame/scan/baseline/contact semantics | reference only; non-commercial hardware license blocks default commercial reuse | not indexed |
| PyFlexiTac | `https://github.com/WT-MM/PyFlexiTac/tree/dc2e1c6607a1b1a0e4b196bb4e2d4ccdc1afdcb0` | MIT | tactile API/normalization semantics | reference only; does not grant hardware/dataset/simulation assets | not indexed |

## Asset distinctions

- **Code:** governed by each repository’s explicit license only.
- **Model/checkpoint:** requires per-model/model-card terms; not implied by framework code license.
- **Dataset:** requires direct dataset-card/license verification; no dataset permission is inferred here.
- **Hardware design:** may use terms distinct from code (e.g., FlexiTac CC BY-NC hardware).
- **Robot/simulator assets:** require their own terms; MANO, Shadow Hand, Nokov, RFUniverse, IsaacSim, and data artifacts are not authorized by this manifest.

No repository was cloned into the EgoGlove product repository, and no third-party implementation was adopted.