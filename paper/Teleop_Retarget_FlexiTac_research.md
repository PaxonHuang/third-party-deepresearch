# Teleoperation / Retargeting / Tactile Sensing Research Distillation

> **Project**: EgoGlove — Human Hand Intelligence Layer for Embodied AI
> **Scope**: AnyDexRetarget (dexterous retargeting), dexbotic (VLA toolkit), FlexiTac (low-cost tactile)
> **Date**: 2026-08-16
> **Author**: Codex research distillation (PaxonHuang review pending)
> **Reality indicators**: ✅ implemented · 🟡 engineering-feasible (6–12 mo) · 🔬 needs R&D · 🌌 long-term

---

## Source Repos & Traceability

| Repo | Path | Commit | Paper |
|------|------|--------|-------|
| AnyDexRetarget | `repo/AnyDexRetarget/` | `fce83d1e564d3e21aa909b3bd6d67c14aee65197` | [AnyTeleop arxiv:2307.04577](https://arxiv.org/pdf/2307.04577) |
| dexbotic | `repo/dexbotic/` | `6356c98e6b75d3f4fbc8765913d64ddfd9fe0823` | [arxiv:2510.23511](https://arxiv.org/pdf/2510.23511) |
| FlexiTac | `repo/FlexiTac/` (Awesome-FlexiTac gallery) | `6c5d111743c49d0fdce90ae28bf2ad56bb6af28c` | [arxiv:2604.28156](https://arxiv.org/abs/2604.28156) |

> **Note on FlexiTac**: The cloned repo is the *Awesome-FlexiTac* curated-paper gallery (`binghao-huang/Awesome-FlexiTac`), not the hardware repo itself. The actual hardware design files are at `github.com/FlexiTac/FlexiTac_Hardware_Repo` (linked but not cloned). Sensor specs below are from the paper abstract on the gallery page; BOM/cost breakdown is **待人工核对** against the hardware repo.

---

# 1. AnyDexRetarget — Universal Hand-to-Robot Retargeting

## 1.1 Overview

AnyDexRetarget is a high-precision hand pose retargeting system mapping human hand keypoints (21-point MediaPipe format) to dexterous robot hand joint angles. It supports **13 robot hands** (Shadow, Wuji, Allegro, Inspire, Ability, Leap, SVH, LinkerHand L21, Linker L20, ROHand, Unitree Dex5, Sharpa, Gaia Hand20) and multiple input sources (Apple Vision Pro, Meta Quest 3, Pico 4, Noitom gloves, MediaPipe camera/video). ✅

**Key claim**: ~2ms per frame real-time performance using analytical gradients + NLopt SLSQP. ✅ (code present, timing instrumented via `TimingStats`)

**Architecture**: Two optimizers — `AdaptiveOptimizerAnalytical` (default, pinch-aware) and `KeyVectorOptimizer` (key-vector matching, referenced as dex-retargeting VectorOptimizer arxiv 2506.11916).

### Repository Structure

```
anydexretarget/
├── retarget.py                  # High-level Retargeter (pipeline orchestrator)
├── robot.py                     # Pinocchio robot wrapper (FK + Jacobian)
├── mediapipe.py                 # MediaPipe → MANO wrist-frame transform
└── optimizer/
    ├── base_optimizer.py        # Base: NLopt setup, mimic joints, link indices
    ├── analytical_optimizer.py  # AdaptiveOptimizerAnalytical (TipDirVec + FullHandVec)
    ├── key_vector_optimizer.py  # KeyVectorOptimizer (generic key-vector matching)
    ├── robot_configs.py         # 13 robot hand link/URDF configs
    └── utils.py                 # Huber loss, LPFilter, TimingStats
example/
├── config/{adaptive,vector}/{mediapipe,avp,quest3,pico4,noitom}/  # YAML configs
├── input/                       # Input device drivers (VisionPro, Quest3, etc.)
├── output/{sim,real}/           # MuJoCo sim + real hardware drivers
└── teleop_real.py               # Full teleoperation entry point
```

Source: [retarget.py](/home/EchoGloveHugeProjects/third-party-deepresearch/repo/AnyDexRetarget/anydexretarget/retarget.py), [base_optimizer.py](/home/EchoGloveHugeProjects/third-party-deepresearch/repo/AnyDexRetarget/anydexretarget/optimizer/base_optimizer.py)

## 1.2 Core Algorithm — Adaptive Optimizer

The adaptive optimizer blends two sub-objectives based on a **pinch distance alpha** that smoothly transitions between open-hand posture matching and pinch-precision fingertip targeting.

### Pinch Alpha Computation

For each non-thumb finger $i$, compute distance to thumb tip:

$$d_i = \|\mathbf{p}_{\text{tip}_i} - \mathbf{p}_{\text{thumb}}\| \cdot 100 \quad (\text{m→cm})$$

$$\alpha_i = \text{clip}\left(\frac{d_2 - d_i}{d_2 - d_1},\; 0,\; 1\right)$$

Where $d_1, d_2$ are per-finger pinch thresholds (default $d_1=2.0$ cm, $d_2=4.0$ cm). $\alpha=1$ → full pinch (TipDirVec mode); $\alpha=0$ → open hand (FullHandVec mode). Thumb alpha = max of all non-thumb alphas.

Source: [analytical_optimizer.py:155-165](/home/EchoGloveHugeProjects/third-party-deepresearch/repo/AnyDexRetarget/anydexretarget/optimizer/analytical_optimizer.py) `_compute_pinch_alpha()`

### Loss Function

$$\mathcal{L}(\mathbf{q}) = \sum_{i=1}^{N_f} \left[\alpha_i \cdot \mathcal{L}_{\text{TipDirVec},i} + \beta_i \cdot \mathcal{L}_{\text{FullHand},i}\right] + \lambda \|\mathbf{q} - \mathbf{q}_{\text{prev}}\|^2$$

Where $\beta_i = (1-\alpha_i) + \alpha_i \cdot w_{\text{pinch\_full\_hand}}$ (convex blend, default $w_{\text{pinch\_full\_hand}}=0$).

**TipDirVec loss** (pinch mode, $\alpha \to 1$):

$$\mathcal{L}_{\text{TipDirVec},i} = w_{\text{pos}} \cdot H_{\delta}(\|\mathbf{v}_{\text{tip}}^{\text{robot}} - \mathbf{v}_{\text{tip}}^{\text{target}}\|) + w_{\text{dir}} \cdot H_{\delta_{\text{dir}}}(\|\hat{\mathbf{d}}_{\text{tip}}^{\text{robot}} - \hat{\mathbf{d}}_{\text{tip}}^{\text{target}}\|)$$

- $\mathbf{v}_{\text{tip}}^{\text{robot}} = \text{FK}_{\text{tip}}(\mathbf{q}) - \text{FK}_{\text{origin}}(\mathbf{q})$ (wrist→tip vector)
- $\hat{\mathbf{d}}_{\text{tip}}^{\text{robot}} = \frac{\text{FK}_{\text{tip}}(\mathbf{q}) - \text{FK}_{\text{DIP}}(\mathbf{q})}{\|\cdot\|}$ (DIP→tip direction)

**FullHandVec loss** (open mode, $\alpha \to 0$):

$$\mathcal{L}_{\text{FullHand},i} = \frac{w_{\text{fh}}}{3} \sum_{s \in \{\text{PIP, DIP, TIP}\}} H_{\delta}(\|\mathbf{v}_{s}^{\text{robot}} - \mathbf{v}_{s}^{\text{target}}\|)$$

All vectors are wrist-relative, scaled by per-finger `segment_scaling[finger][PIP, DIP, TIP]`.

$H_{\delta}$ is Huber loss:

$$H_{\delta}(x) = \begin{cases} \frac{1}{2}x^2 & |x| \le \delta \\ \delta(|x| - \frac{\delta}{2}) & |x| > \delta \end{cases}$$

Source: [analytical_optimizer.py:296-390](/home/EchoGloveHugeProjects/third-party-deepresearch/repo/AnyDexRetarget/anydexretarget/optimizer/analytical_optimizer.py) `_loss_and_grad_analytical()`

### KeyVectorOptimizer (Alternative)

Generic key-vector matching, minimizing mean Huber distance between robot vectors and scaled human vectors:

$$\mathcal{L}(\mathbf{q}) = \frac{1}{N} \sum_{i=1}^{N} H_{\delta}(\|[\text{FK}(\text{task}_i) - \text{FK}(\text{origin}_i)] - s_i \cdot [\text{mp}(\text{task\_kp}_i) - \text{mp}(\text{origin\_kp}_i)]\|) + \lambda \|\mathbf{q} - \mathbf{q}_{\text{prev}}\|^2$$

Each key vector is defined by an `(origin_link, task_link)` pair with optional local offsets. This is the more general/flexible formulation — any set of link pairs can be matched to any keypoint pairs.

Source: [key_vector_optimizer.py](/home/EchoGloveHugeProjects/third-party-deepresearch/repo/AnyDexRetarget/anydexretarget/optimizer/key_vector_optimizer.py)

## 1.3 Optimization Pipeline

1. **Input**: Raw keypoints `(21, 3)` from any input source
2. **Coordinate transform**: `apply_mediapipe_transformations()` — wrist-centered, MANO frame via SVD-based frame estimation + `OPERATOR2MANO` rotation matrix ✅
3. **Optional rotation adjustment**: Extrinsic XYZ Euler angles (config `mediapipe_rotation`)
4. **Robot-specific preprocessing**: Wrist/thumb offset correction, optional MCP alignment to robot kinematics
5. **Pinch alpha computation** → adaptive target vectors
6. **NLopt SLSQP**: `maxeval=50`, `ftol_abs=1e-4`, analytical gradients provided
7. **Low-pass filter**: `y = y + α(x - y)`, default `α=0.4`

### Mimic Joint Handling ✅

URDF `<mimic>` joints are parsed and handled:
- Independent joints are the optimization variables (`num_opt_vars`)
- Mimic joints expanded via `expand_to_full_qpos()`: `q[mimic] = q[source] * multiplier + offset`
- Gradient mapped back via chain rule through `map_gradient_to_independent()`
- Joint limits applied only to independent joints

Source: [base_optimizer.py:265-363](/home/EchoGloveHugeProjects/third-party-deepresearch/repo/AnyDexRetarget/anydexretarget/optimizer/base_optimizer.py) `_parse_mimic_joints()`, `expand_to_full_qpos()`

### Joint Limit Handling ✅

- URDF joint limits from Pinocchio model (`lowerPositionLimit`, `upperPositionLimit`)
- Config override `clamp_joint_lower` to prevent hyperextension (e.g., Shadow Hand J3 clamped to ≥0.0)
- NLopt SLSQP enforces box constraints on independent joints

### Forward Kinematics ✅

Uses **Pinocchio** (`pinocchio` Python bindings):
- `pin.buildModelFromUrdf()` → model
- `pin.forwardKinematics()` + `pin.updateFramePlacements()` for batch FK
- `pin.computeJointJacobians()` + `pin.getFrameJacobian()` for batch Jacobians
- Position Jacobian with local offset: `J_point = J[:3,:] - [offset]_× @ J[3:,:]` (skew-symmetric correction)

Source: [robot.py](/home/EchoGloveHugeProjects/third-party-deepresearch/repo/AnyDexRetarget/anydexretarget/robot.py) `RobotWrapper`

## 1.4 Data Structure — Config Format

YAML config per robot+input combination. Example (Shadow Hand + MediaPipe):

```yaml
# example/config/adaptive/mediapipe/mediapipe_shadow_hand.yaml
optimizer:
  type: "AdaptiveOptimizerAnalytical"
robot:
  type: "shadow_hand"
retarget:
  huber_delta: 2.0           # Position Huber threshold (cm)
  huber_delta_dir: 0.5       # Direction Huber threshold
  norm_delta: 0.04           # Velocity regularization weight
  w_pos: 1.0                 # Tip position weight
  w_dir: 5.0                 # Tip direction weight
  scaling: 0.81              # MP→Shadow scale (MP ~24% larger)
  w_full_hand: 1.0
  segment_scaling:           # Per-finger [PIP, DIP, TIP]
    thumb:  [0.884, 0.883, 0.854]
    index:  [1.079, 1.03, 1.028]
    # ...
  pinch_thresholds:
    index:  { d1: 2.0, d2: 4.0 }   # cm
  mediapipe_rotation: { x: -10.0, y: 0.0, z: -120.0 }
  clamp_joint_lower:
    "J3": 0.0                 # Prevent MCP hyperextension
  lp_alpha: 0.4
```

Source: [mediapipe_shadow_hand.yaml](/home/EchoGloveHugeProjects/third-party-deepresearch/repo/AnyDexRetarget/example/config/adaptive/mediapipe/mediapipe_shadow_hand.yaml)

### Robot Config Registry

`robot_configs.py` maps each robot type to link names:

```python
ROBOT_CONFIGS = {
    'shadow_hand': {
        'origin_link': 'rh_palm',
        'tip_links': ['rh_thtip', 'rh_fftip', ...],     # 5 fingertips
        'link1_names': ['rh_thproximal', ...],           # proximal phalanx
        'link3_names': ['rh_thmiddle', ...],             # middle/PIP
        'link4_names': ['rh_thdistal', ...],             # distal/DIP
        'urdf_subdir': 'assets/shadow_hand',
        'urdf_file': {'right': 'right_hand_mj.urdf', 'left': 'left_hand_mj.urdf'},
        'num_fingers': 5,
    },
    'gaia_hand20': { 'num_fingers': 5, 'neutral_qpos': [0.0]*20, ... },
    'allegro_hand': { 'num_fingers': 4, ... },           # no pinky
    # ... 13 hands total
}
```

Source: [robot_configs.py](/home/EchoGloveHugeProjects/third-party-deepresearch/repo/AnyDexRetarget/anydexretarget/optimizer/robot_configs.py)

## 1.5 Key API Signatures

```python
class Retargeter:
    @classmethod
    def from_yaml(cls, yaml_path: str, hand_side: str = "right") -> "Retargeter"
    
    def retarget(self, raw_keypoints: np.ndarray, apply_filter: bool = True) -> np.ndarray:
        """(21, 3) raw keypoints → (num_joints,) joint angles"""
    
    def retarget_verbose(self, raw_keypoints, apply_filter=True) -> Tuple[np.ndarray, dict]:
        """Returns (qpos, {mediapipe_kp, qpos_unfiltered, qpos, cost, pinch_alphas})"""

class BaseOptimizer(ABC):
    def solve(self, mediapipe_keypoints: np.ndarray, last_qpos: Optional[np.ndarray] = None) -> np.ndarray
    def compute_cost(self, qpos: np.ndarray, mediapipe_keypoints: np.ndarray) -> float

class RobotWrapper:
    def compute_points_batch(self, qpos, link_indices, local_offsets=None) -> np.ndarray
    def compute_all_jacobians_batch_with_offsets(self, qpos, link_indices, local_offsets=None) -> np.ndarray
    @property
    def joint_limits(self) -> np.ndarray  # (nq, 2) [lower, upper]
```

## 1.6 Engineering Pitfalls

1. **Scaling calibration is manual** ✅: Each robot requires per-finger `segment_scaling` and global `scaling` tuned by hand. The `debug_skeleton.py` tool shows blue (raw) / green (scaled) / red (FK) skeletons side-by-side for calibration. No auto-calibration. 🟡
2. **MediaPipe-only input format** ✅: The pipeline expects 21-point MediaPipe keypoints. Other input sources (VisionPro, Quest3, Noitom) are normalized to this format by their respective input drivers. The coordinate transform uses SVD-based frame estimation from wrist+MCP landmarks, which can be noisy.
3. **Pinch thresholds are robot-dependent** ✅: Default $d_1=2.0, d_2=4.0$ cm work for human-scale hands but need adjustment for smaller/larger robot hands.
4. **No collision avoidance**: The optimizer does not check self-collision or object collision. Joint limits are the only safety constraint. 🟡
5. **NLopt SLSQP can fail silently**: `RuntimeError` from NLopt is caught and falls back to `init_qpos`, which may produce jerky motion if optimization frequently fails. ✅ (code handles this)
6. **Left-hand support is per-robot hack**: Link name prefix replacement (`rh_`→`lh_`, `right_`→`left_`, `R`→`L`) is hardcoded per robot type in multiple places. Adding a new robot requires adding a new branch. 🟡
7. **Mimic joint parsing depends on URDF**: If URDF mimic tags are missing or incorrect, the optimizer will treat mimic joints as independent, potentially over-parameterizing. ✅
8. **`lateral_scaling` compression** 🟡: Anisotropic palm compression (only Y-axis) is a Gaia-specific workaround. The `preserve_pinch_lateral` flag gradually removes compression as fingers pinch, but this is an empirical heuristic, not a principled solution.

---

# 2. dexbotic — Open-Source VLA Toolkit

## 2.1 Overview

Dexbotic is a unified VLA (Vision-Language-Action) development toolbox supporting pretraining, fine-tuning, inference, and evaluation. It supports mainstream policies: π0, π0.5, CogACT, OFT, MemVLA, DM0, GR00T-N1, DiscreteVLA, NaVILA, UniNaVid, and MuVLA. ✅

**Architecture**: "Layered config + factory registration + entry dispatch". Modular policy wrappers (`BasePolicy` → `Pi0Policy`, `DiscreteVLAPolicy`, etc.) with unified `select_action()` interface.

Source: [README.md](/home/EchoGloveHugeProjects/third-party-deepresearch/repo/dexbotic/README.md), [base_policy.py](/home/EchoGloveHugeProjects/third-party-deepresearch/repo/dexbotic/dexbotic/policy/base_policy.py)

## 2.2 Data Format — Video + JSONL

### Dataset Layout

```
data/libero/libero_spatial/
├── video/                    # data_path_prefix: MP4 video files
│   ├── episode_001/
│   │   ├── images_1/         # Multi-view image sequences (or video)
│   │   ├── images_2/
│   │   └── ...
│   └── ...
├── *.jsonl                   # annotations: per-frame metadata
└── index_cache.json          # Auto-built index: {total_samples, total_jsonl_files, data}
```

Source: [libero_official.py](/home/EchoGloveHugeProjects/third-party-deepresearch/repo/dexbotic/dexbotic/data/data_source/libero_official.py), [dex_dataset.py:319-340](/home/EchoGloveHugeProjects/third-party-deepresearch/repo/dexbotic/dexbotic/data/dataset/dex_dataset.py)

### JSONL Episode Format

Each `.jsonl` file contains one JSON object per line (per frame). Fields include:

```json
{
  "prompt": "pick up the red mug",
  "state": [0.1, 0.2, ...],
  "images_1": [{"type": "video", "path": "episode_001/cam_0.mp4", "frame": 42}],
  "images_2": [{"type": "image", "path": "episode_001/cam_1/0042.jpg"}],
  "is_robot": true
}
```

The `LoadMultiModal` transform handles both video (via `decord.VideoReader` or `av`) and image loading, indexed by `frame_indicies`. ✅

Source: [multimodal.py](/home/EchoGloveHugeProjects/third-party-deepresearch/repo/dexbotic/dexbotic/data/dataset/transform/multimodal.py)

### Conversation Format (for LLM-style training)

```json
{"from": "human", "value": "<image>\nWhat action should the robot take to pick up the red mug?"}
{"from": "gpt",  "value": " 127 42 0 198 55 12 0"}
```

The human turn contains the `<image>` token + task prompt; the GPT turn contains the discretized action string.

Source: [language.py](/home/EchoGloveHugeProjects/third-party-deepresearch/repo/dexbotic/dexbotic/data/dataset/transform/language.py) `ToConversation`, [default_transform.py](/home/EchoGloveHugeProjects/third-party-deepresearch/repo/dexbotic/dexbotic/data/dataset/transform/default_transform.py)

## 2.3 Action Tokenization — 256-Bin Discretization

### Normalization

Actions are normalized to $[-1, 1]$ using per-dataset, per-prompt min/max statistics:

$$a_{\text{norm}} = \text{clip}\left(\frac{a - \min}{\max - \min + \epsilon} \cdot 2 - 1,\; -1,\; 1\right)$$

The `statistic_mapping` supports hierarchical overrides: `default` → `dataset` → `prompt`.

Source: [action.py:350-380](/home/EchoGloveHugeProjects/third-party-deepresearch/repo/dexbotic/dexbotic/data/dataset/transform/action.py) `ActionNormAnd2String`

### Discretization (256-bin)

$$b = \text{round}\left(\frac{a_{\text{norm}} + 1}{2} \cdot (V - 1)\right), \quad \text{clip}(b, 0, V-1)$$

Where $V = 256$ (default `vocab_size=255` in some policies, 256 in the `ToConversationWithDiscreteState` class which uses `np.linspace(-1, 1, 256+1)` bins). The discrete token is then converted to a string:

```python
action_str = ''.join([f" {int(b)}" for b in bin_action])  # e.g. " 127 42 0 198 ..."
```

**Decoding** (at inference):

$$a_{\text{norm}} = \frac{b}{V-1} \cdot 2 - 1$$

$$a = \frac{a_{\text{norm}} + 1}{2} \cdot (\max - \min + \epsilon) + \min$$

Source: [discrete_vla_arch.py](/home/EchoGloveHugeProjects/third-party-deepresearch/repo/dexbotic/dexbotic/model/discrete_vla/discrete_vla_arch.py) `_discrete_action_to_continuous()`, `_denorm()`

### State Discretization (optional)

`ToConversationWithDiscreteState` discretizes robot state into the prompt:

```python
discretized_state = np.digitize(state, bins=np.linspace(-1, 1, 256+1)[:-1]) - 1
state_str = " ".join(map(str, discretized_state))
prompt = f"Task: {prompt}, State: {state_str}"
```

Source: [language.py:145-165](/home/EchoGloveHugeProjects/third-party-deepresearch/repo/dexbotic/dexbotic/data/dataset/transform/language.py)

### Normalization Statistics (norm_stats.json)

For continuous-action policies (π0, DM0, etc.), statistics are stored as `NormStats`:

```python
@dataclass
class NormStats:
    mean: NDArray
    std: NDArray
    q01: NDArray | None    # 1st percentile
    q99: NDArray | None    # 99th percentile
    min: NDArray | None
    max: NDArray | None
```

Computed via `RunningStats` with online histogram-based quantile estimation (5000 bins). ✅

Source: [normalize.py](/home/EchoGloveHugeProjects/third-party-deepresearch/repo/dexbotic/dexbotic/data/utils/normalize.py)

## 2.4 Model Architecture — Policy Families

### A-Family (Discrete VLA)

- `DiscreteVLAPolicy`: Action tokens decoded autoregressively by LLM `generate()`
- `Gr00tN1Policy`: Flow-matching action head with `cfg_scale` + `num_ddim_steps`
- Supports chat templates: `dexbotic`, `step`, `qwen2-chat`

### B-Family (Continuous VLA)

- `Pi0Policy`: Flow-matching with state conditioning, `action_dim=7`, `num_images=3`
- `Pi05Policy`: π0.5 variant (no state consumption by action model)
- `DM0Policy`: DM0 flow-matching
- `OFTPolicy`: OFT transformer
- `CogactPolicy`, `MemVlaPolicy`, `Gr00tSonicPolicy`: Additional model wrappers

Inference: `model.inference_action(input_ids, images, inference_args)` → `[B, chunk, action_dim]`

Source: [pi0_policy.py](/home/EchoGloveHugeProjects/third-party-deepresearch/repo/dexbotic/dexbotic/policy/pi0_policy.py), [discrete_vla_policy.py](/home/EchoGloveHugeProjects/third-party-deepresearch/repo/dexbotic/dexbotic/policy/discrete_vla_policy.py)

## 2.5 Inference API

### HTTP v1 Protocol

```python
# POST /v1/infer
{
  "observation": {
    "prompt": "pick up the red mug",
    "images": {"1": "<base64 PNG>", "2": "<base64 PNG>"},
    "state": [0.1, 0.2, ...]     # optional
  },
  "sampling": {"num_steps": 10, "cfg_scale": 1.5, "seed": null}
}
# Response: {"actions": [[...], ...]}   # [chunk, action_dim]
```

### Python Client

```python
class DexClient:
    def act(self, observation: dict, prompt: str) -> list:
        """Returns next action from queue (auto-refills from server)"""
    
    def reset(self) -> None:
        """Clear action queue + notify server"""
```

Delta action mode: `action[6:] = 0; action += delta; wrap rotation dims to [-π, π]`

Source: [client.py](/home/EchoGloveHugeProjects/third-party-deepresearch/repo/dexbotic/dexbotic/client.py)

### Hardware Bridge (SO-101 example)

gRPC-based bridge: `lerobot` transport → `BridgeService` → `DexClient` HTTP → VLA server. Action chunking with queue. ✅

Source: [bridge_server.py](/home/EchoGloveHugeProjects/third-party-deepresearch/repo/dexbotic/hardware/so101/bridge_server.py)

## 2.6 Key API Signatures

```python
class BasePolicy(ABC):
    action_mode: str          # "absolute" | "relative"
    state_used: bool
    state_required: bool
    state_dim: Optional[int]
    max_batch_size: int = 1
    
    @abstractmethod
    def select_action(self, observation: dict, sampling_config: Optional[SamplingConfig] = None) -> list[ActionOutput]
    
    def reset(self) -> None
    def supports_vlm(self) -> bool
    def get_capabilities(self) -> dict

@dataclass
class ActionOutput:
    actions: np.ndarray  # [chunk_size, action_dim], physical units, no batch dim

@dataclass
class SamplingConfig:
    num_steps: int = 10       # flow matching / DDIM steps
    cfg_scale: float = 1.5
    seed: Optional[int] = None

class ActionNormAnd2String:
    def __init__(self, statistic_mapping: dict, vocab_size: int = 255, 
                 string_format: str = ' {value}', add_answer: bool = True)
```

## 2.7 Engineering Pitfalls

1. **vocab_size inconsistency** ✅: `ActionNormAnd2String` defaults to `vocab_size=255`, but `ToConversationWithDiscreteState` uses 256 bins. `DiscreteVLAPolicy` defaults to `vocab_size=255`. The decode formula `actions / (vocab_size - 1) * 2 - 1` means 255→[0,254]→[-1,1], 256→[0,255]→[-1,1]. Must match between encode and decode. 🟡
2. **Error handling masks data issues** ✅: `DexDataset.__getitem__` catches all exceptions and returns a random sample, silently hiding data corruption.
3. **Action dimension mismatch** ✅: `PadState`/`PadAction` pad to 32-dim by default, but `action_dim` varies (7 for LIBERO, 6 for SO-101). The `non_delta_mask` must align with actual action structure.
4. **No native dexterous hand support** ✅: All examples are arm+gripper (7-DOF: 6 pose + 1 gripper). Dexterous hand actions (20+ DOF) would need custom `statistic_mapping`, `action_dim`, and potentially new policy wrappers. 🔬
5. **Video loading via decord/av** ✅: Both `decord.VideoReader` and `av` are imported; if neither is available, video data loading fails. Image-only mode is more robust.
6. **Index cache can go stale** ✅: `_check_index_cache` only checks jsonl file count, not content. If a jsonl file is modified (samples added/removed) without adding/removing files, the cache is stale.
7. **DiscreteVLA retry loop** ✅: `inference_action` retries up to 40 times on exception, which can mask persistent errors and cause long timeouts.

---

# 3. FlexiTac — Low-Cost Piezoresistive Tactile Array

## 3.1 Overview

FlexiTac is a low-cost, open-source, scalable piezoresistive tactile sensing solution for robotic end-effectors. Key features from the paper abstract:

- **Three-layer laminate**: FPC-Velostat-FPC stack with electrode patterns integrated into flexible printed circuits ✅ (paper claim, 待人工核对 against hardware repo)
- **100 Hz multi-channel readout** via serial communication ✅ (paper claim)
- **Compact readout board** with low-cost components ✅ (paper claim)
- **Multiple form factors**: fingertip pads + larger tactile mats ✅ (paper claim)
- **3D visuo-tactile fusion** for contact-aware decision making 🔬 (demonstrated in 3D-ViTac follow-up)
- **Cross-embodiment skill transfer** 🔬 (paper claim)
- **Real-to-sim-to-real** with GPU-parallel tactile simulation 🔬 (paper claim)

> **Cost**: The task brief mentions "$30 sensor". The paper abstract does not specify exact cost. The related STAG glove (Nature 2019, same group lineage) claims ~$10 for a 548-sensor glove. FlexiTac's $30 claim is **待人工核对** against the hardware repo BOM.

Source: [flexitac.html abstract](/home/EchoGloveHugeProjects/third-party-deepresearch/repo/FlexiTac/papers/flexitac.html), [arxiv:2604.28156](https://arxiv.org/abs/2604.28156)

## 3.2 Hardware Topology

> **Note**: The following is reconstructed from the paper abstract + related works (3D-ViTac, STAG). Detailed schematics, ADC part numbers, and exact electrode pitch are in the hardware repo (`github.com/FlexiTac/FlexiTac_Hardware_Repo`) which is **not cloned**. All hardware specifics below are **待人工核对**.

### Sensor Stack (Cross-Section)

```
┌─────────────────────────────────┐
│  FPC Top Layer (electrodes)     │  ← Flexible printed circuit
├─────────────────────────────────┤
│  Velostat piezoresistive film   │  ← Force-sensitive resistive layer
├─────────────────────────────────┤
│  FPC Bottom Layer (electrodes)  │  ← Orthogonal electrode pattern
└─────────────────────────────────┘
         ↓ Pressure
   Row × Column → taxel at intersection
```

### Readout Architecture (inferred)

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Tactile Pad │     │  Tactile Pad │     │  Tactile Pad │
│  (N×M taxels)│     │  (N×M taxels)│     │  (N×M taxels)│
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │ FPC ribbon          │                    │
       ▼                    ▼                    ▼
┌──────────────────────────────────────────────────────┐
│           Multi-Channel Readout Board                 │
│  ├── MUX (row/col scanning)                          │
│  ├── ADC (analog → digital)                          │
│  ├── MCU (serial protocol, 100 Hz)                   │
│  └── USB/Serial → Host PC                            │
└──────────────────────────────────────────────────────┘
       │ Serial @ 100 Hz
       ▼
   Host PC (tactile learning pipeline)
```

### Cost Breakdown (待人工核对)

| Component | Estimated Cost | Source |
|-----------|---------------|--------|
| FPC fabrication (electrode pattern) | ~$5-10 | 待人工核对 |
| Velostat sheet | ~$1-2 | 待人工核对 |
| Readout board (MCU + MUX + ADC) | ~$10-15 | 待人工核对 |
| Connectors/cabling | ~$2-3 | 待人工核对 |
| **Total (per fingertip pad)** | **~$20-30** | 待人工核对 |

> The $30 figure is plausible given the STAG glove precedent (~$10 for 548 sensors using conductive thread). FPC fabrication is more expensive than conductive thread but more repeatable.

## 3.3 3D Visuo-Tactile Fusion (from 3D-ViTac)

The related 3D-ViTac system (same research group, featured in the Awesome-FlexiTac gallery) demonstrates the visuo-tactile fusion approach:

- **Tactile sensor specs**: Dense sensing units, each covering 3mm² ✅ (3D-ViTac paper)
- **3D representation space**: Tactile and visual data fused into a unified 3D representation preserving spatial relationships ✅ (paper claim)
- **Diffusion policy integration**: Multi-modal representation coupled with diffusion policies for imitation learning ✅ (paper claim)
- **Application**: Bimanual dexterous manipulation, fragile object handling, long-horizon in-hand manipulation 🔬

Source: [3d-vitac.html](/home/EchoGloveHugeProjects/third-party-deepresearch/repo/FlexiTac/papers/3d-vitac.html)

## 3.4 Data Processing (待人工核对)

> The actual data processing code is in the hardware repo, not the gallery repo. Based on the paper abstract and related works:

- Serial stream at 100 Hz → host PC parses multi-channel pressure array
- Each frame: `N_taxels × pressure_value` matrix
- Temporal smoothing and calibration likely required (待人工核对)
- Integration with tactile simulation (GPU-parallel) for sim-to-real 🔬

## 3.5 Engineering Pitfalls

1. **Velostat variability** 🟡: Piezoresistive film (Velostat/Linqstat) has significant batch-to-batch and sensor-to-sensor variability. Calibration per-pad is likely required.
2. **Hysteresis** 🟡: Piezoresistive sensors exhibit hysteresis (different response for loading vs unloading). This is a known limitation of Velostat-based sensors.
3. **Wear and drift** 🟡: Velostat degrades with repeated loading. Long-term use requires periodic recalibration.
4. **FPC design is non-trivial** 🟡: Electrode pattern design (pitch, trace width, insulation) affects sensitivity and spatial resolution. The FPC approach improves repeatability vs conductive thread but requires PCB design skills.
5. **100 Hz may be limiting** 🟡: For high-speed dynamic manipulation, 100 Hz update rate may be insufficient. Vision systems typically run at 30-60 Hz, but tactile events can be faster.
6. **No cloned hardware code**: The gallery repo contains only the paper collection website. All hardware, firmware, and data processing code is in a separate repo not available here. All hardware claims are **待人工核对**.

---

# 4. Cross-Source Comparison: Retargeting Approaches

## 4.1 AnyDexRetarget vs. DexCap-style Retargeting

AnyDexRetarget's README references `dex-retargeting VectorOptimizer (arxiv 2506.11916)`. The DexCap paper (arxiv 2403.07788, also in the paper directory) is the origin of the `dex-retargeting` library. AnyDexRetarget extends this approach.

| Dimension | DexCap / dex-retargeting | AnyDexRetarget |
|-----------|------------------------|-----------------|
| **Optimizer** | NLopt SLSQP + autograd | NLopt SLSQP + **analytical gradients** ✅ |
| **Loss** | Vector matching (key vectors) | Adaptive blend: TipDirVec + FullHandVec ✅ |
| **Pinch handling** | None (uniform weighting) | Pinch-aware alpha blending ✅ |
| **Robots supported** | ~6 (Allegro, Shadow, etc.) | **13** (Shadow, Wuji, Allegro, Inspire, Ability, Leap, SVH, Linker L21, Linker L20, ROHand, Dex5, Sharpa, Gaia) ✅ |
| **Input sources** | MediaPipe | MediaPipe + VisionPro + Quest3 + Pico4 + Noitom ✅ |
| **Mimic joints** | Limited | Full URDF mimic parsing + gradient mapping ✅ |
| **Performance** | ~5-10ms (autograd) | **~2ms** (analytical gradients) ✅ |
| **Config** | Python dicts | YAML configs per robot+input ✅ |
| **Link offsets** | Fixed | Per-link local offsets with skew-symmetric Jacobian correction ✅ |
| **Lateral scaling** | No | Anisotropic palm compression (Gaia-specific) 🟡 |

## 4.2 Mapping to EgoGlove's Robot Action Layer

EgoGlove's architecture (from `docs/V7/ARCHITECTURE.md`) defines a four-stage pipeline:

```
Sensor Source → Hand Token v2 (canonical-20) → Skeleton Layer (20-rotation, FK→21 points) → {MANO Layer, Robot Action Layer}
```

The Robot Action Layer is currently at 🔬 (待接, D12 P1). AnyDexRetarget provides a ready implementation path:

### Direct Integration Points

1. **Hand Token v2 → AnyDexRetarget input**: AnyDexRetarget expects 21-point MediaPipe keypoints `(21, 3)`. Hand Token v2's canonical-20 skeleton FK-derives 21 points. The `apply_mediapipe_transformations()` function already handles the wrist-frame transform. An adapter at the Skeleton Layer output can feed directly into `Retargeter.retarget()`. 🟡

2. **Canonical-20 ↔ MediaPipe-21 alignment**: AnyDexRetarget uses MediaPipe landmark indices: `MP_ORIGIN_IDX=0` (wrist), `MP_TIP_INDICES=[4,8,12,16,20]`, `MP_PIP_INDICES=[2,6,10,14,18]`, `MP_DIP_INDICES=[3,7,11,15,19]`. EgoGlove's FK-derived 21 points must match this indexing. The canonical-20 rotation joints need to produce positions at these exact landmark locations. 🟡

3. **Multi-robot support**: AnyDexRetarget already supports 13 robot hands. EgoGlove's D12 goal of "开放手部运动基础设施" aligns directly. Adding a new robot hand requires only: (a) URDF in `assets/`, (b) config entry in `robot_configs.py`, (c) YAML config file. 🟡

4. **Real-time constraint**: AnyDexRetarget's ~2ms per frame is well within EgoGlove's teleoperation latency budget. The analytical gradient approach avoids autograd overhead. ✅

### Gaps to Address

| Gap | Priority | Approach |
|-----|----------|----------|
| EgoGlove Hand Token v2 → 21-point adapter | P0 | FK from canonical-20 → 21 MediaPipe-indexed positions; verify coordinate frame match |
| Retargeting config per target robot | P1 | Generate YAML configs for EgoGlove's target robots (likely start with Inspire/Gaia for domestic, Shadow for research) |
| Calibration auto-tuning | P2 | Replace manual `segment_scaling` with automatic calibration from hand-size measurement |
| Dexterous hand + arm VLA integration | 🔬 | dexbotic's VLA pipeline handles arm+gripper (7-DOF); extending to 20+ DOF dexterous hands requires custom action tokenization, norm_stats, and policy wrappers |
| Tactile feedback loop | 🌌 | FlexiTac on robot fingertips → tactile data → VLA conditioning for contact-aware manipulation (long-term) |

### dexbotic + EgoGlove Robot Action Layer

dexbotic's action format is currently arm+gripper oriented. For EgoGlove's dexterous hand use case:

1. **Action representation**: EgoGlove's Robot Action Layer outputs joint angles (e.g., 20-DOF for canonical-20). dexbotic's `ActionNormAnd2String` can handle arbitrary dimensions via `statistic_mapping`, but the 256-bin discretization may lose precision for 20+ DOF. 🟡
2. **Data collection**: dexbotic's video+JSONL format is directly usable for dexterous hand data collection. The `state` field would carry 20-DOF joint angles; `action` would carry the next-step target. ✅
3. **Policy training**: Flow-matching policies (π0, DM0) handle continuous actions better than discrete tokenization for high-DOF hands. The B-family policies are more suitable. 🟡
4. **Retargeting as preprocessing**: AnyDexRetarget can convert EgoGlove Hand Token → robot joint angles offline, generating training data for dexbotic VLA models. This separates the teleoperation pipeline (real-time retargeting) from the learning pipeline (offline VLA training). ✅

### FlexiTac + EgoGlove

FlexiTac is complementary to EgoGlove rather than directly integrated:
- EgoGlove captures **human hand motion** (the operator side)
- FlexiTac captures **robot hand tactile feedback** (the robot side)
- Together: human intent (EgoGlove) → robot action (retargeting) → tactile feedback (FlexiTac) → VLA learning (dexbotic) 🔬

This maps to EgoGlove's long-term vision of closed-loop teleoperation with tactile feedback, but is clearly 🌌 for the first generation.

---

# 5. Summary Table

| Feature | AnyDexRetarget | dexbotic | FlexiTac |
|---------|---------------|----------|----------|
| **Domain** | Hand retargeting | VLA training/inference | Tactile sensing |
| **Maturity** | ✅ Production-ready | ✅ Toolkit-ready | ✅ Paper + hardware repo |
| **EgoGlove relevance** | P0 (Robot Action Layer) | P1 (data collection + VLA) | P2 (tactile feedback) |
| **Key algorithm** | Adaptive TipDirVec + FullHandVec | 256-bin action discretization | Piezoresistive array + 3D fusion |
| **Real-time** | ✅ ~2ms/frame | 🟡 Inference-dependent | ✅ 100 Hz |
| **Cost** | Software (free) | Software (free) | ~$30/sensor (待人工核对) |
| **Integration effort** | Low (adapter at Skeleton Layer) | Medium (dexterous hand action format) | High (hardware + firmware) |
| **Unverified claims** | None significant | vocab_size consistency | Cost, BOM, sensor specs (待人工核对) |

---

## References

- AnyTeleop paper: [arxiv:2307.04577](https://arxiv.org/pdf/2307.04577)
- dex-retargeting (DexCap): [arxiv:2506.11916](https://arxiv.org/abs/2506.11916), [arxiv:2403.07788](https://arxiv.org/abs/2403.07788)
- dexbotic paper: [arxiv:2510.23511](https://arxiv.org/pdf/2510.23511)
- FlexiTac paper: [arxiv:2604.28156](https://arxiv.org/abs/2604.28156)
- FlexiTac hardware repo: [github.com/FlexiTac/FlexiTac_Hardware_Repo](https://github.com/FlexiTac/FlexiTac_Hardware_Repo)
- 3D-ViTac: featured in Awesome-FlexiTac gallery
- EgoGlove architecture: `docs/V7/ARCHITECTURE.md` (D3, D10, D11, D12)
