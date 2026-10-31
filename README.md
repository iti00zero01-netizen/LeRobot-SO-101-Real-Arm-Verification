# LeRobot SO-101 Real-Arm Verification Project
This repository serves as a technical verification log for setting up environment, calibrating hardware, collecting datasets, and troubleshooting policy training using **LeRobot (ACT Model)** with real SO-101 robot arms on a **Quadro P4000 GPU (Pascal architecture / sm_61)** under WSL2 environment.
---
## System Architecture
- **Host OS:** Windows Pro (WSL2 Architecture)
- **Guest OS:** Ubuntu 22.04 LTS / 24.04 LTS
- **Package Manager:** Miniconda3 (Python 3.12 / Virtual Environment: `lerobot`)
- **Compute Unit:** NVIDIA Quadro P4000 (sm_61, Pascal generation)
---
## Execution Pipeline (Step-by-Step)
### 1. Host Side: Windows USB Pass-through
Connect real hardware controllers and split USB cameras directly into independent physical ports to distribute bandwidth, then share them into WSL2 via PowerShell:
```powershell
winget install dresden-elektronik.wsl-usbipd
usbipd list
usbipd attach --wsl --busid <YOUR_BUSID>
```
### 2. Guest Side: Virtual Env & Custom Core Installation
Downgrade PyTorch pinpointing to `2.4.1` to maintain `sm_61` binary execution compatibility while avoiding standard dependency overrides via `--no-deps`:
```bash
source ~/.bashrc
conda activate lerobot
git clone https://github.com
cd lerobot
pip install torch==2.4.1 torchvision==0.19.1 torchaudio==2.4.1 --extra-index-url https://pytorch.org
pip install --no-deps -e ".[dynamixel]"
pip install --no-deps -e ".[dataset]"
pip install opencv-python
```
### 3. Camera Verification & Privilege Release
```bash
sudo apt update && sudo apt install -y v4l-utils
sudo modprobe uvcvideo
ls /dev/video*
sudo usermod -aG video \$USER
sudo chmod 666 /dev/video*
sudo apt update && sudo apt install -y fswebcam
fswebcam -d /dev/video0 --no-banner test_cam1.jpg
```
### 4. Interactive Arm Alignment Calibration
```bash
ls -l /dev/ttyACM*
python lerobot/scripts/lerobot_find_port.py
lerobot-calibrate --teleop.type=so101_leader --teleop.port=/dev/ttyACM1 --teleop.id=my_leader_arm
lerobot-calibrate --teleop.type=so101_follower --teleop.port=/dev/ttyACM0 --teleop.id=my_follower_arm
```
### 5. Live Teleoperation
```bash
python src/lerobot/scripts/lerobot_teleoperate.py \
--robot.type=so101_follower \
--robot.port=/dev/ttyACM0 \
--robot.id=my_follower_arm \
--teleop.type=so101_leader \
--teleop.port=/dev/ttyACM1 \
--teleop.id=my_leader_arm
```
### 6. Dataset Recording (50 Episodes)
```bash
python lerobot/scripts/record.py \
--robot.type=so101 \
--robot.leader_port=/dev/ttyACM1 \
--robot.follower_port=/dev/ttyACM0 \
--repo_id="your_dataset_name_perfect_final" \
--num_episodes=50 \
--warmup_time_s=5 \
--episode_time_s=30 \
--reset_time_s=5 \
--device=cuda \
--wandb.enable=false
```
### 7. Offline Policy Training Trigger (Encountered Conflict)
```bash
python \$(find . -name "train_policy.py" -o -name "train.py" | head -n 1) \
--dataset.repo_id="your_dataset_name_perfect_final" \
--policy.type=act \
--env.type=real \
--device=cuda \
--training.offline_steps=20000 \
--training.save_checkpoint_every=5000 \
--training.log_every=100 \
--eval.n_episodes=0 \
--wandb.enable=false
```
---
## Summary of Findings (Technical Insight)
1.**Bandwidth Challenges:** When attempting to capture video for a dataset using two USB cameras—one positioned for a close-up view of the manipulator and the other for an overhead view—we encountered issues due to the cameras supporting only the YUYV format rather than MJPEG. This resulted in dropped frames and the appearance of "green screens," likely caused by compression processing. Although we implemented countermeasures, such as connecting each camera directly to a separate physical port rather than through a single USB hub, the problem persisted. Consequently, we will need to replace these units with new USB cameras.
2. **Framework Bottleneck:** While hardware control and image capturing succeeded, triggering offline policy training under the current repo master failed due to structural import level clashes between `torch 2.4.1` (the last compiler supporting `sm_61`) and newer `torch.accelerator` abstraction layer modules added in the recent version of LeRobot. Future mitigation requires checkout targeting an older stable release branch.
