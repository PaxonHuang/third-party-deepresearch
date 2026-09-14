# 11. Multimodal Pose Architecture Lessons

> Status: 🔬 source-grounded third-party research distillation; not an EgoGlove implementation claim.
> Updated: 2026-09-07
> Purpose: bounded evidence for Demo2A/Demo2B architecture review.
> Isolation: upstream source snapshots remain in `third-party-deepresearch/repo/`, are not copied into EgoGlove, and are not included in the EgoGlove CodeBaseMemory/graph index.

## 1. Evidence and license discipline

This document distinguishes:

- **Source-supported fact:** directly observed in pinned upstream source, README, license, or official documentation.
- **Project inference:** an architectural implication for EgoMotion, not an upstream promise.
- **AI engineering advice:** kept separately in `advicebyAI/`; it is not source fact.
- **Unverified hypothesis:** requires local fixture, hardware, or data.

Repository code, model/checkpoint, dataset, hardware design, simulation, and robot assets are separately licensed. Visibility of a GitHub repository or paper never grants commercial reuse.

## 2. Source-supported lessons

### 🔬 FSGlove — calibration is a first-class system problem

**Pinned source:** `https://github.com/davidliyutong/fsglove/tree/a7988f051c35f5ee6157353ee4022d73aa1e4cc8`, 2026-05-05. Repository contains no root `LICENSE`/`COPYING`/`NOTICE`; code reuse permission is therefore **not established**.

**Source-supported fact:** README describes inertial hand tracking with IMUs on finger joints/dorsum, up to 48 DoF, and a DiffHCal approach that jointly considers kinematics, shape parameters, and sensor misalignment. Its protocol preserves sensor ID, accel/gyro/mag, quaternion, pressure, device ticks, timestamp, sequence, and validity.

**Source-supported fact:** FSGlove is a materially different 16-IMU system with optional optical/Vive wrist references and host-side MANO-related calibration. Its reported project-page numbers are not EgoMotion measurements.

**Source-supported license fact:** MANO assets must be downloaded under the MANO license; that license is non-commercial/non-redistributable for the stated model/data/sample-code terms. NOKOV, RFUniverse, Shadow Hand, datasets, and other visualizer assets require separate verification.

**Project inference:** sensor-to-bone extrinsic, morphology, joint-axis assumptions, and optional external root reference must remain separate calibration artifacts; a board axis is not an anatomical axis.

### 🔬 Kalibr — camera/IMU spatial-temporal calibration categories

**Pinned source:** `https://github.com/ethz-asl/kalibr` at local snapshot `1f60227442d25e36365ef5f72cd80b9666d73467`; BSD 3-clause-style repository license.

**Source-supported fact:** Kalibr documents multi-camera intrinsics/extrinsics, camera–IMU spatial and temporal calibration with IMU intrinsics, IMU–IMU spatial/temporal calibration with camera aid, and rolling-shutter intrinsic calibration including shutter parameters.

**Project inference:** a camera branch must preserve intrinsic/distortion/readout model, `T_I^C`, temporal offset, calibration evidence, residual, and validity interval. Flexible wearable mounting/re-donning validity is not solved merely by invoking a camera–IMU calibration tool.

### 🔬 GTSAM — factor graph framing is not an EgoMotion solver contract

**Pinned source:** `https://github.com/borglab/gtsam` at local snapshot `8be77851e956f2087600950ac1b936e0d340440c`; BSD repository license.

**Source-supported fact:** GTSAM describes smoothing/mapping with factor graphs and Bayes networks; its documented generic workflow is build factor graph, linearize/solve in tangent spaces, retract to manifolds, iterate.

**Evidence limitation:** the reviewed material does not establish an EgoMotion fixed-lag/sliding-window policy, factor residuals, calibration gates, or acceptance metrics.

**Project inference:** factor names alone are insufficient; residual frame/dimension, covariance units/order, time association, calibration dependencies, robust loss, and failure gate must be explicit before any solver choice.

### 🔬 manotorch / MANO — downstream parametric projection only

**Pinned source:** `https://github.com/lixiny/manotorch` at local snapshot `a2a70c591f91551078b7bb2af9b5d9f275b626e0`; repository labelled Apache-2.0.

**Source-supported fact:** manotorch describes a differentiable mapping from hand pose/shape parameters to vertices, joints, and transforms. Its README describes 778 vertices, 21 joints, 16 articulation transforms, root-relative meters, and required separately downloaded MANO assets.

**License boundary:** repository code terms do not grant MANO model/data terms. MANO must be registered/downloaded separately and is not a commercial/re-distributable default dependency.

**Project inference:** MANO is optional projection/fitting data with its own fit residual and provenance; it cannot define Canonical Human Hand State.

### 🔬 hand-tracking-fusion-system — a baseline prototype, not accuracy evidence

**Pinned source:** `https://github.com/alexandertianlin/hand-tracking-fusion-system/tree/165f40d0b7593884514eb80a68c58bb455637d43`, 2026-07-14; MIT code repository.

**Source-supported fact:** README describes a RealSense/vision plus six-IMU/tactile glove prototype, per-finger confidence gating, orientation gating, a Slerp complementary fusion path, and open-palm calibration.

**Evidence limitation:** the same README states MPJPE, PA-MPJPE, and drift-reduction evaluation are pending. Claimed latency/thresholds are self-described prototype settings, not transferable performance evidence.

**Project inference:** a Demo2A Slerp baseline may be useful only as an explicitly bounded baseline. Its thresholds, calibration routine, sensor count, camera model, and output quality cannot become canonical semantics.

### 🔬 xio Fusion — AHRS failure/bias semantics deserve first-class representation

**Pinned source:** `https://github.com/xioTechnologies/Fusion/tree/9325424011892abacc0ce42b8bb1a8ae20264b9b`, tag `v1.3.3`, 2026-08-30; MIT.

**Source-supported fact:** this embedded C/Python AHRS provides quaternion, gravity, linear/earth acceleration; documented settings include gyro+accel with optional magnetometer/external heading, startup ramp, acceleration/magnetic rejection, overrange recovery, stationary bias estimation, NWU/ENU/NED conventions, and 24 orthogonal axis remaps. It states that intrinsic/magnetic calibration parameters must be provided externally.

**Project inference:** gyro/accelerometer bias, scale/misalignment, axis remap, sample-period error, rejection/overrange/startup state, and temperature/range validity must live in calibration/quality/state contracts. AHRS output is a derived orientation observation, not hand joint truth.

### 🔬 MediaPipe — vision is a measurement producer, not canonical truth

**Pinned source:** `https://github.com/google-ai-edge/mediapipe/tree/c17b2a83e8944d2811889a2a08d629c20bcb6ed8`; Apache-2.0 code.

**Source-supported fact:** MediaPipe provides packet/graph/calculator-oriented cross-platform ML infrastructure and task/model workflows.

**License boundary:** model binaries, model cards, training data, and web/demo assets require independent verification; the repository license alone is not enough.

**Project inference:** landmark position/confidence/visibility/occlusion is a camera observation with image/exposure/inference timing and source-profile provenance. It is not a global wrist pose or Canonical State by default.

### 🔬 FlexiTac — tactile is exteroceptive contact evidence

**Pinned sources:** paper `https://arxiv.org/abs/2604.28156` (paper page CC BY 4.0); hardware `https://github.com/FlexiTac/FlexiTac_Hardware_Repo` at `e006b179e89b79be28dd7062c5dd75ed866e5e7a` (CC BY-NC 4.0); PyFlexiTac `https://github.com/WT-MM/PyFlexiTac` at `dc2e1c6607a1b1a0e4b196bb4e2d4ccdc1afdcb0` (MIT). IsaacSim repository-wide license and dataset/asset terms remain unverified.

**Source-supported fact:** upstream describes flexible piezoresistive FPC–Velostat–FPC tactile modules; hardware documents a `16×32` raw frame with `AA 55` header plus 512 `uint8` samples and a 100 Hz serial stream. PyFlexiTac describes `read() -> FlexiTacFrame(seq, timestamp_s, raw, normalized)` and a per-pixel median baseline over initial frames; its runtime defaults may differ from hardware dimensions.

**License boundary:** CC BY-NC hardware is not a default commercial reuse path. MIT PyFlexiTac does not grant hardware, dataset, robot, or simulation asset rights.

**Project inference:** tactile raw frame geometry, scan time, baseline/normalization, saturation, spatial calibration, and contact-region mapping must be retained. Derived contact/pressure/centroid/normal are not joint-angle truth.

## 3. Architecture consequences supported by the sources

1. IMU/camera spatial-temporal calibration needs explicit parameter, residual, uncertainty, validity, and mounting assumptions. Source: Kalibr; wearable validity is a project inference.
2. An estimator cannot be described only by its solver name. Factor/state/measurement/covariance/quality contracts are portable; a particular library is not. Source: GTSAM/xio Fusion; contract conclusion is project inference.
3. IMU-derived orientation needs bias/rejection/overrange/startup/timing semantics. Source: xio Fusion.
4. Vision and tactile branches are measurements with confidence/quality/time/profile metadata; neither implies canonical global pose or joint truth. Source: MediaPipe/FlexiTac/hand-tracking-fusion-system.
5. Sensor-to-bone/morphology/misalignment calibration is not optional. Source: FSGlove; transfer to EgoMotion remains a project inference requiring local validation.
6. MANO must remain a separately licensed downstream projection. Source: manotorch/MANO license.

## 4. Explicit gaps and prohibited claims

- No reviewed source proves EgoMotion’s 5+1 or 10+1 accuracy, latency, calibration repeatability, global wrist pose, or observability rank.
- No reviewed source licenses all of its code, models, datasets, hardware, and robot assets under one reusable commercial term.
- No source justifies treating host receive time as source event time.
- No source justifies treating tactile/flex as absolute joint-angle truth.
- No source makes an extra IMU independently informative without distinct rigid-segment placement, calibration, and timing evidence.

## 5. Local relationship

This research informs the architecture gate in `EgoGlove/docs/superpowers/specs/2026-09-07-demo2a-multimodal-pose-architecture-second-review.md`. It does not claim any source has been adopted or implemented by EgoGlove.
