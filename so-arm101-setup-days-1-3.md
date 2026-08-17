# SO-ARM101 Standard Kit — Days 1–3 on an M1 Max MacBook Pro

Adapted from the official Hiwonder user manual (Windows/Ubuntu only) to macOS Apple Silicon.
Kit: **Standard Kit / Pre-Assembled** — one 300K-pixel camera, arms already built, servo IDs already set.

Goal by end of Day 3: leader arm drives follower arm, cameras stream, one demonstration episode
recorded and replayed.

---

## Read this first

### Buy one thing

A **1080p USB webcam**, ~$20–30. Your kit has one 300K-pixel (480p) camera. Every command in the
manual assumes two: a `handeye` camera at the gripper and a `front`/`fixed` camera watching the
workspace. Use the kit camera on the gripper bracket and the new webcam as the fixed view.

Do **not** buy a USB hub — a 4-port, 1000 mm hub is included in the Standard Kit.

### Get the Hiwonder resource bundle

The manual constantly references paths like `2.Software Tools & Source Code\Source Code\lerobot.zip`
and `1. Tutorials\Video Tutorials\3.2 Hardware Assembly`. **You need that bundle** — it contains the
LeRobot source they've pinned and the assembly/wiring videos. Find the download link on
`https://www.hiwonder.com.cn/store/learn/185.html` or in the kit's paperwork. Do this before Day 1.

### The macOS situation

The manual has no macOS path. But their software is standard LeRobot with the standard Feetech motor
driver, so the only real difference is port naming: where the manual says `COM22` / `COM24`, you use
`/dev/tty.usbmodemXXXX`. Everything else should carry over.

**Time-box it.** Give macOS one focused day. If the environment won't build or the ports won't open
by end of Day 1, switch to a Windows or x86 Linux machine rather than burning the project on it.
A virtual machine on your Mac is not a good fallback — USB camera passthrough into VMs is unreliable
and you'd end up debugging virtualization instead of robotics.

---

## Day 1 — Physical setup and ports

### 1.1 Workspace and safety

Clamp both arms to the desk with the included G-clamps. An unclamped arm drags itself around and
damages the printed parts.

The follower's HX-30HM servos produce 30 kg·cm of stall torque at a 1:345 reduction. That pinches
hard. The manual's own warning: keep the area around the arm clear, and `Ctrl+C` in the terminal is
your abort. Keep the power plug within reach and never leave the arm powered while you walk away.

**Follow the manual's wiring order:** mechanical structure → camera → servo cables → USB cables →
power. Disconnect power before touching any servo cable. Hot-plugging servos causes both erratic
motion and port-recognition failures.

Watch the assembly, camera, and circuit-connection videos in the Hiwonder bundle (sections 3.1–3.3).
Your arms are pre-assembled, so this is mostly wiring and mounting.

### 1.2 Prerequisites

```bash
xcode-select --install
```

Miniconda, **arm64 build**:

```bash
curl -O https://repo.anaconda.com/miniconda/Miniconda3-latest-MacOSX-arm64.sh
bash Miniconda3-latest-MacOSX-arm64.sh
```

Restart the terminal, then confirm you are not running under Rosetta:

```bash
python -c "import platform; print(platform.machine())"
```

Must print `arm64`. If it prints `x86_64`: quit your terminal app, find it in Finder, Get Info,
uncheck "Open using Rosetta", reopen.

### 1.3 Identify the ports

The manual's Ubuntu section shows the boards appearing as `/dev/ttyACM0` and `/dev/ttyACM1`, which
means they enumerate as standard USB CDC devices. On macOS that maps to `/dev/tty.usbmodem*` with
**no driver installation needed**.

With nothing connected:

```bash
ls /dev/tty.*
```

Connect the **follower** arm first (matching the manual's order), then run it again. Note the new
entry. Then connect the **leader** and run it again.

```
FOLLOWER (COM24 in the manual) = /dev/tty.usbmodem________
LEADER   (COM22 in the manual) = /dev/tty.usbmodem________
```

Write these down. The manual warns repeatedly not to reverse them.

⚠️ On macOS these names can change when you replug or use a different port. If commands suddenly
fail later, re-run `ls /dev/tty.*` before assuming anything else is wrong.

If nothing appears at all, that is your signal to switch machines rather than debug further.

### 1.4 Power

Two 12V 5A adapters (DC 5.5×2.5) — one per arm. Confirm the power LEDs. Do not command anything yet.

**Day 1 is done when:** both arms are clamped and wired, and both show up under `/dev/tty.*`.

---

## Day 2 — Environment, calibration, teleoperation

### 2.1 Create the environment

Exactly as the manual specifies (they pin these versions deliberately — 3.10, not 3.12):

```bash
conda create -n lerobot python=3.10.18 ffmpeg=7.1.1 -c conda-forge
```

Expect 5–20 minutes. Press `y` when prompted. Then:

```bash
conda activate lerobot
```

You must re-run `conda activate lerobot` in every new terminal window.

> If `ffmpeg=7.1.1` has no build for osx-arm64, relax it to `ffmpeg=7.1` or install ffmpeg separately
> with `conda install ffmpeg -c conda-forge`. The version matters because it carries the `libsvtav1`
> encoder that the dataset format uses.

### 2.2 Install their LeRobot

Extract `lerobot.zip` from the Hiwonder bundle to your Desktop, then:

```bash
cd ~/Desktop/lerobot
pip install -e ".[feetech]"
```

Expect 3–10 minutes. If this step fails with macOS-specific dependency errors, that is decision
point two: switch machines.

### 2.3 Calibrate

Skip servo ID setup entirely — section 3.5 applies only to DIY kits, and yours is pre-assembled.

Calibration teaches the software the correspondence between raw encoder values and joint positions,
so the leader and follower agree on what a given pose means.

**Before running the command, put the arm in the calibration start pose.** The manual shows photos on
pages 25 (leader) and 26 (follower) — open the PDF to those pages and match them. The leader pose is
the arm folded out flat and horizontal, gripper handle hanging down at one end, base at the other.

Leader:

```bash
lerobot-calibrate \
  --teleop.type=so101_leader \
  --teleop.port=/dev/tty.usbmodemYOURLEADER \
  --teleop.id=my_awesome_leader_arm
```

Follower:

```bash
lerobot-calibrate \
  --robot.type=so101_follower \
  --robot.port=/dev/tty.usbmodemYOURFOLLOWER \
  --robot.id=my_awesome_follower_arm
```

Press Enter to begin. **If you are recalibrating something already calibrated, type `c` first, then
Enter** — otherwise it reuses the old values. Then manually rotate each joint through its range as
prompted. There is a follower calibration video in the bundle.

Keep the `--teleop.id` and `--robot.id` strings identical in every later command. The manual uses
`my_awesome_leader_arm` and `my_awesome_follower_arm`; keeping their names means you can copy their
commands verbatim.

### 2.4 Teleoperate — no cameras yet

The manual is emphatic about doing this before touching cameras, so that you are debugging one
subsystem at a time.

```bash
python -m lerobot.teleoperate \
  --robot.type=so101_follower \
  --robot.port=/dev/tty.usbmodemYOURFOLLOWER \
  --robot.id=my_awesome_follower_arm \
  --teleop.type=so101_leader \
  --teleop.port=/dev/tty.usbmodemYOURLEADER \
  --teleop.id=my_awesome_leader_arm
```

Press Enter at the prompt. Move the leader slowly by hand; the follower mirrors it. `Ctrl+C` to stop.

**If direction or position is wrong on any joint:** per the manual's FAQ 8.3, return both arms to the
initial pose and recalibrate both, entering `c` when prompted.

**Day 2 is done when:** the follower tracks the leader smoothly with correct directions on all six joints.

---

## Day 3 — Cameras and first recording

### 3.1 Camera permission

macOS blocks camera access per app and gives no error — you get black frames. Run the camera command
once so your terminal appears in the list, then go to **System Settings → Privacy & Security → Camera**
and enable it.

### 3.2 Mount and find the cameras

Kit 300K camera → gripper bracket (this is the `handeye` view).
Your new webcam → fixed position covering the whole workspace (this is `front`).

The manual requires that the fixed camera capture the follower arm's **full motion range**.

```bash
lerobot-find-cameras opencv
```

Takes 10–60 seconds. Test images land in `outputs/captured_images`. Open them: `opencv_0` corresponds
to `index_or_path: 0`, `opencv_1` to `1`. Your built-in FaceTime camera will also appear — make sure
you know which index is which.

⚠️ The manual warns explicitly that **camera indices are not stable** across replugging, port
changes, or reboots. Re-run this command any time you reconnect USB devices.

⚠️ Also from the manual: do not connect both cameras to the same hub or dock. Put at least one
directly into a Thunderbolt port on the Mac.

### 3.3 Teleoperate with vision

```bash
python -m lerobot.teleoperate \
  --robot.type=so101_follower \
  --robot.port=/dev/tty.usbmodemYOURFOLLOWER \
  --robot.id=my_awesome_follower_arm \
  --robot.cameras="{ front: {type: opencv, index_or_path: 1, width: 640, height: 480, fps: 30}, handeye: {type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30}}" \
  --teleop.type=so101_leader \
  --teleop.port=/dev/tty.usbmodemYOURLEADER \
  --teleop.id=my_awesome_leader_arm \
  --display_data=true
```

Substitute your actual indices.

**The camera names become dataset feature names.** Once you start recording real data, the number of
cameras, their names, and their index mapping must stay identical through recording, training, and
inference. Pick `handeye` and `front` now and never change them.

**Freeze the physical setup too.** Tape the camera mounts down. Fix the lighting — close the blinds
and use consistent artificial light, because sun through a window is a variable the policy will
learn and then be confused by. If a camera shifts halfway through data collection, the earlier half
of your dataset is wasted.

### 3.4 Record two throwaway episodes

Not real data yet — just proving the pipeline works. Substitute an English username for `Anabelle`.

```bash
python -m lerobot.record \
  --robot.type=so101_follower \
  --robot.port=/dev/tty.usbmodemYOURFOLLOWER \
  --robot.id=my_awesome_follower_arm \
  --robot.cameras="{ handeye: {type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30}, front: {type: opencv, index_or_path: 1, width: 640, height: 480, fps: 30}}" \
  --teleop.type=so101_leader \
  --teleop.port=/dev/tty.usbmodemYOURLEADER \
  --teleop.id=my_awesome_leader_arm \
  --display_data=true \
  --dataset.repo_id=Anabelle/test_throwaway \
  --dataset.num_episodes=2 \
  --dataset.single_task="Put the block in the cup" \
  --dataset.push_to_hub=false
```

Startup takes 20–60 seconds. Then the keyboard controls, from the manual:

| Key | Action |
|---|---|
| Enter | Continue past a prompt |
| **→** | Finish the demonstration / finish the reset / save |
| **←** | Discard this episode and re-record it |
| **Esc** | Stop recording entirely |

The cycle per episode is: demonstrate → press → → reset the scene → press → → wait for the save
progress bar to reach 100%.

Data lands in `~/.cache/huggingface/lerobot`.

**Do not unplug USB or close the terminal during saving.**

### 3.5 Confirm it recorded

Open `~/.cache/huggingface/lerobot/Anabelle/test_throwaway` and play the recorded videos. Confirm
both camera streams contain actual images and not black frames. This is the whole point of the
throwaway run.

**Day 3 is done when:** two episodes exist on disk with non-black video from both cameras.

---

## Days 4–7 — Real data

Delete the throwaway dataset. Then record **20 episodes** of one task (the manual's default; increase
later if the policy is weak). Budget 30–90 minutes.

The manual's guidance, which is worth following exactly:

- Practise the motion by teleoperation until you can do it smoothly *before* recording anything.
  Hesitation and repeated corrections get baked into the policy.
- Start every episode from the same standardised initial arm position.
- Vary the object's placement between episodes — do not put it in the identical spot each time.
- Keep everything except the arm static in frame. No hands, no people, no drifting objects.
- Use a stable, repeatable grasping style. The policy learns your habits, including the bad ones.

---

## Days 8+ — Training (cloud, not your Mac)

The manual states plainly that local training without a discrete NVIDIA GPU is not recommended, and
that an RTX 4090 needs 2–3 hours for the default 100,000 steps. Your M1 Max has no CUDA. Plan on a
rented GPU: Colab, Lambda, RunPod, or similar.

Workflow: push the dataset to the Hugging Face Hub from your Mac → train on the cloud GPU → download
the checkpoint → run inference locally on the arm.

Training command shape (on the GPU machine):

```bash
python src/lerobot/scripts/train.py \
  --dataset.repo_id=Anabelle/demo \
  --policy.type=act \
  --output_dir=outputs/train/act_so101_test \
  --job_name=act_so101_test \
  --policy.device=cuda \
  --wandb.enable=false \
  --policy.push_to_hub=false
```

Useful knobs in `src/lerobot/configs/train.py`: `steps` (default 100,000), `batch_size` (default 8),
`save_freq` (default every 20,000 steps). Reducing `steps` is the honest way to trade quality for
turnaround time on a first attempt.

Inference runs back on your Mac via `lerobot.record` with `--policy.path=...` and no leader arm.
Use `checkpoints/last/pretrained_model` if you changed the step count.

**Known bug from the manual's FAQ:** the second inference run fails because the eval output directory
already exists. Delete the `eval_*` folder manually between runs.

---

## Debugging principle

Change one thing at a time. Port problems, calibration problems, camera-index problems, and dependency
problems all produce similarly vague failures. If you change three things and it starts working, you
have learned nothing and it will break again.

Quick triage from the manual's FAQ:

| Symptom | First thing to check |
|---|---|
| `conda` not recognised | Reopen the terminal |
| Arm won't connect | Re-run `ls /dev/tty.*` — the name may have changed |
| Wrong direction on a joint | Recalibrate both arms, typing `c` at the prompt |
| Camera missing or wrong view | Re-run `lerobot-find-cameras opencv`; don't put both cameras on the hub |
| Policy behaves erratically | Too few episodes, or object position never varied during collection |
| Model path not found | The number in `checkpoints/100000/` must match your `steps` setting |
