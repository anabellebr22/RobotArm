# SO-ARM101 Imitation Learning Project

## What this is

A two-week personal learning project: teach a SO-ARM101 robotic arm to do one
pick-and-place task by imitation learning, using Hugging Face's LeRobot framework.

Target task: pick up a 1-inch colored wooden cube from a marked area and drop it in a cup.
Deliberately narrow. Success means the full loop works end to end — calibrate, teleoperate,
record demonstrations, train an ACT policy, run it autonomously.

This is for fun and learning, not for coursework or a lab application.

## Who I am

Incoming MS student at NYU Tandon (Emerging Technologies, robotics concentration).
Strong computer science background. **No robotics or hardware background.**

How to help me:
- Explain hardware and robotics concepts in full, plain detail.
- **Do not explain robotics concepts by analogy to programming.** Explain the actual thing.
- Assume I can read and write code fine; assume I know nothing about servos, torque,
  calibration, kinematics, or control.
- When something fails, help me isolate one variable at a time rather than changing several
  things at once.
- **Give me one task at a time.** Long batched instructions have caused you to stall before
  taking any action. Small scoped requests, then stop and report.
- Do not tell me something works if you haven't verified it from files or command output.
  If you're inferring, say you're inferring.

## Do not modify the lerobot source

The `lerobot/` directory is vendored third-party code from Hiwonder's distribution.
Do not edit files inside it. If something in lerobot appears to be the problem, say so
and explain — don't patch it. Debug my configuration, ports, and commands first.

---

## Hardware

- **Kit:** Hiwonder SO-ARM101, Standard Kit, pre-assembled
- **Leader arm** (black): 6× HX-10HM bus servos, 1:147 reduction, 10 kg·cm stall torque.
  Low reduction so it drags smoothly by hand. This is the input device.
- **Follower arm** (white): 6× HX-30HM bus servos, 1:345 reduction, 30 kg·cm stall torque.
  This is the execution end. **This is the one that can pinch — 30 kg·cm is not a toy.**
- **Controller:** BusLinker V3.0 servo debugging board, one per arm, USB serial
- **Power:** 12V 5A adapter per arm (DC 5.5×2.5)
- **Joints** top to bottom: gripper, wrist_roll, wrist_flex, elbow_flex, shoulder_lift,
  shoulder_pan = servo IDs 6 down to 1
- **Servo IDs are already set** — the pre-assembled kit does not need `lerobot-setup-motors`

### Cameras
- Kit camera: 300K pixel (480p) — mounted on the gripper, named `handeye` in commands
- Logitech C270 (fixed focus, USB-A) — fixed environment view, named `front` in commands
- Both record at 640×480 @ 30fps

### Workspace
- Black foam board work surface, arms clamped with G-clamps
- 1-inch wooden cubes as the target object (single color for the first dataset)

### USB port plan (Mac has 3 Thunderbolt ports, 4 devices)
- Port 1 — one arm board, direct via Type-C
- Port 2 — Logitech C270 via USB-C→USB-A adapter
- Port 3 — adapter → kit's 4-port hub → other arm board + kit camera

Manual requires at least one camera direct to the computer, and both cameras must not
share a hub.

---

## Computer and environment

**2021 MacBook Pro, M1 Max, macOS.**

⚠️ **The official Hiwonder manual only covers Windows and Ubuntu.** No macOS path exists in
their docs. Every command in their manual uses `COM22` / `COM24` and must be translated to
`/dev/tty.usbmodem*` here. The repo does contain a `requirements-macos.txt`, so some macOS
support exists in the code even though the manual ignores it.

⚠️ **No CUDA.** Training ACT locally is not viable. The manual states an RTX 4090 needs 2–3
hours for the default 100,000 steps. Plan: record data locally → push dataset to Hugging Face
Hub → train on a rented cloud GPU (Colab/Lambda/RunPod) → download checkpoint → run inference
locally on the arm. Inference is a small model at 30 Hz and runs fine on CPU.

### WORKING environment — verified, do not rebuild casually

```bash
conda activate lerobot
cd ~/Desktop/RobotArm/lerobot
```

Built with:
```bash
conda create -n lerobot python=3.10.18 ffmpeg=7.1.1 -c conda-forge
conda activate lerobot
pip install "av<15"                  # REQUIRED before the line below — see gotchas
pip install -e ".[feetech]"
```

Verified working state:

| Check | Value |
|---|---|
| `platform.machine()` | `arm64` |
| conda | `~/miniconda3` (Miniconda **arm64**) |
| Python | 3.10.18 |
| `lerobot.__file__` | `~/Desktop/RobotArm/lerobot/src/lerobot/__init__.py` (editable install OK) |
| `av.__version__` | 14.4.0 |
| `cv2.__version__` | 5.0.0 (built from source, arm64) |
| `torch.__version__` | 2.7.1 |
| `torch.backends.mps.is_built()` | True |
| `torch.backends.mps.is_available()` | **False** — open question, see below |

---

## Environment gotchas (hard-won — read before touching the environment)

1. **`/opt/anaconda3` is an Intel (x86_64) build. NEVER use it for this project.**
   It reports `__osx==10.16` (the version macOS reports to Intel processes) and produces
   x86_64 environments even when asked for arm64. This caused the first install to fail:
   pip fetched x86_64 wheels, and `torchvision>=0.21` has no Intel Mac build, so the
   dependency set was unsatisfiable. `CONDA_SUBDIR=osx-arm64` does **not** fix it.
   The fix was installing Miniconda arm64 to `~/miniconda3`.

2. **ALWAYS verify architecture before running any pip command:**
   ```bash
   python -c "import platform; print(platform.machine())"
   ```
   Must print `arm64`. This single check would have saved an hour.

3. **`av` must be installed as `"av<15"` BEFORE `pip install -e ".[feetech]"`.**
   The version LeRobot resolves to by default has no arm64 wheel, compiles from source,
   and fails. Version 14.4.0 has a prebuilt wheel and satisfies the `>=14.2.0` constraint.

4. **A Homebrew FFmpeg at `/opt/homebrew/Cellar/ffmpeg/7.0.2` shadows the conda FFmpeg
   during native builds.** This is what broke the `av` source build — it compiled against
   Homebrew's 7.0.2 headers instead of conda's 7.1.1. If a future build errors against
   FFmpeg headers, this is why. Workaround if needed:
   `PKG_CONFIG_PATH="$CONDA_PREFIX/lib/pkgconfig" pip install <package>`

5. **`opencv-python-headless` 5.0.0.93 has no arm64 wheel** and compiles from source
   (roughly 30–60 minutes). It does succeed. If rebuilding and impatient,
   `pip install "opencv-python-headless<5"` gets a prebuilt 4.x wheel that satisfies the
   `>=4.9.0` constraint.

6. **OPEN QUESTION: MPS reports built=True but available=False** on arm64 torch 2.7.1.
   Not blocking — training happens on a cloud GPU, inference runs on CPU. If investigating
   later, first check `sw_vers -productVersion`; MPS requires macOS 12.3 or newer.
   **Do not reinstall torch to chase this.** The environment took hours to get working.

---

## Port and camera identification

Ports are NOT stable on macOS across replugs. Re-check whenever anything is reconnected.

```bash
ls /dev/tty.*
lerobot-find-port
lerobot-find-cameras opencv    # camera indices also shift between sessions
```

Record current values here as they're discovered:

```
LEADER   port = /dev/tty.usbmodem________
FOLLOWER port = /dev/tty.usbmodem________
handeye camera index = ___
front   camera index = ___
```

---

## Command reference

Identifiers must stay consistent across calibrate / teleoperate / record / inference.

**Calibrate** (put the arm in the pose shown on manual pages 25–26 first; type `c` first if
recalibrating something already calibrated):
```bash
lerobot-calibrate --teleop.type=so101_leader --teleop.port=LEADER --teleop.id=my_awesome_leader_arm
lerobot-calibrate --robot.type=so101_follower --robot.port=FOLLOWER --robot.id=my_awesome_follower_arm
```

**Teleoperate** (do this without cameras first, always):
```bash
python -m lerobot.teleoperate \
  --robot.type=so101_follower --robot.port=FOLLOWER --robot.id=my_awesome_follower_arm \
  --teleop.type=so101_leader --teleop.port=LEADER --teleop.id=my_awesome_leader_arm
```

Add cameras with:
```
--robot.cameras="{ handeye: {type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30}, front: {type: opencv, index_or_path: 1, width: 640, height: 480, fps: 30}}" --display_data=true
```

**Record** — same as teleoperate plus `--dataset.repo_id=`, `--dataset.num_episodes=20`,
`--dataset.single_task="..."`, `--dataset.push_to_hub=false`.
Keys during recording: right-arrow = finish/save, left-arrow = discard and re-record,
Esc = stop. Data lands in `~/.cache/huggingface/lerobot`.

**Train** (on a CUDA machine, not this Mac):
```bash
python src/lerobot/scripts/train.py --dataset.repo_id=USER/demo --policy.type=act \
  --output_dir=outputs/train/act_so101_test --job_name=act_so101_test \
  --policy.device=cuda --wandb.enable=false --policy.push_to_hub=false
```

**Inference** — `python -m lerobot.record` with `--policy.path=` and no leader arm.

---

## Operational gotchas

- **Camera permission:** macOS silently returns black frames until the terminal is granted
  camera access in System Settings → Privacy & Security → Camera. No error is shown.
- **Camera indices shift** after any USB replug. Always re-run `lerobot-find-cameras`.
- **Do not put both cameras on the same hub.** At least one must go directly into a
  Thunderbolt port.
- **Inference second-run bug:** the `eval_*` output directory must be deleted manually
  between inference runs or the run fails on a path conflict.
- **Checkpoint path:** `checkpoints/100000/` must match the `steps` value used in training.
  Use `checkpoints/last/` if steps was changed.
- **Never hot-plug servo cables.** Power down first.
- **Camera position and lighting must be frozen** once real data collection starts. A camera
  that shifts mid-dataset invalidates the earlier episodes.

## Safety

The follower arm produces 30 kg·cm of torque and can snap to a position on power-up with no
warning. Keep the power plug reachable, keep the area clear, Ctrl+C aborts. Never leave it
powered and unattended.

---

## About ACT (the policy being trained)

Action Chunking with Transformers (Zhao et al., 2023, the ALOHA paper). Key points:
- Predicts a chunk of ~100 future actions from one observation, rather than one action at a
  time. This cuts compounding error by shrinking the number of decisions per episode.
- Temporal ensembling: queried every timestep anyway, then overlapping chunk predictions for
  the current timestep are averaged with exponentially decaying weights, for smooth motion
  that still reacts to new camera input.
- Trained as a conditional VAE so that contradictory human demonstrations aren't averaged
  into mush. The latent is set to zeros at inference.
- Architecture: ResNet-18 per camera, then tokens, then a transformer encoder-decoder,
  producing an output matrix of (chunk length × 6 joints).
- **Outputs are target joint positions, not torques.** The servos' internal controllers
  handle low-level tracking.

---

## Current status

- [x] Environment built and verified working (arm64, Python 3.10.18, editable LeRobot install)
- [ ] Serial ports identified
- [ ] Arms clamped and wired
- [ ] Calibration
- [ ] Teleoperation without cameras
- [ ] Cameras mounted and found
- [ ] Teleoperation with vision
- [ ] 20 demonstration episodes recorded
- [ ] ACT trained on cloud GPU
- [ ] Autonomous inference

Next step: run `lerobot-find-port` with both arms wired, and fill in the port table above.
