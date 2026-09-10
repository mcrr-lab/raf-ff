# RAF-FF

A ROS 1 package that uses **Kinova Gen3 (6-DOF)** arm with a **Robotiq 2F-140** gripper and a
wrist-mounted **Intel RealSense** camera to feed finger foods to a user. Also pick up a cup and hold it at a sip pose.

Uses multiple AI models to complete the feeding

| Model | Purpose | 
| --- | --- | 
| **GPT-4o-mini** | To identify food on the plate / classify each item's size and firmness 
| **DINO-X 1.0 + SAM 2.1** | Locate and segment the selected item → mask for grasping 
| **Gemini 2.0 Flash Live** | Voice interface — the user asks for a drink or a specific food (in a [separate React app](https://github.com/mahanthesh0r/Gemini_voice_interface))

---

## 1. Setup

### Hardware
Kinova Gen3 (6 DOF, Robotiq 2F-140) at `192.168.1.10` · Intel RealSense on the end effector ·
NVIDIA GPU with CUDA (SAM 2 runs locally).


### Software
ROS 1 (Noetic) · Python 3 · CUDA-enabled PyTorch.

ROS packages: `kortex_driver`, `kortex_description`, `realsense2_camera`, `cv_bridge`, `tf`, `tf2_ros`,
`message_filters`, `xacro`.

Python: `torch`, `torchvision`, `opencv-python`, `numpy`, `scipy`, `scikit-learn`, `pillow`,
`supervision`, `sam2` ([SAM 2](https://github.com/facebookresearch/sam2), plus a local
[Grounded-SAM-2](https://github.com/IDEA-Research/Grounded-SAM-2) checkout),
`dds-cloudapi-sdk` (DINO-X), `requests`, `python-dotenv`, `pyyaml`, `colorlog`, `pygame`,
`SpeechRecognition`, `openai-whisper`.

### Keys
Put your keys in `scripts/.env`:

```
OPENAI_API_KEY = '...'
DINOX_API_KEY  = '...'
```

---

## 2. Important files

| File | Role |
| --- | --- |
| `scripts/cam_detection.py` | ROS node / entry point — starts the feeding loop |
| `scripts/inference_class_final.py` | Core logic: GPT food ID, DINO-X + SAM 2 segmentation, grasp selection, bite transfer |
| `scripts/prompts/identification.txt` | GPT prompt — available foods, single/multi-bite category, firmness |
| `scripts/config/config.yaml` | Runtime config (autonomy, plate depth, offsets, transfer joint poses) |
| `scripts/robot_controller/robot_controller.py` | Kinova joint / pose / gripper commands and named poses |
| `scripts/skill_library.py` | Maps measured food width → gripper value |
| `scripts/raf_utils.py` | Grasp geometry, firmness → grip mapping, TF helpers |
| `scripts/vision_utils.py` | Mask cleanup, centroid and grasp-point extraction |
| `scripts/rs_ros.py` | RealSense color/depth subscriber |
| `src/controller.cpp` | C++ Kortex bridge exposing the pose/joint/gripper services |
| `launch/feed.launch` | Brings up the controller, RealSense, and Kortex driver |

---

## 3. Running

Build once:

```bash
cd ~/catkin_ws && catkin_make && source devel/setup.bash
```

Then, in two terminals:

```bash
roslaunch raf_v3 feed.launch      # robot + camera + controller
rosrun raf_v3 cam_detection.py    # feeding loop
```

Optional, for voice control — set `voice_command: true` in `config.yaml` and run:

```bash
rosrun raf_v3 voice_command.py
```

Useful `config.yaml` knobs: `AUTONOMY` (false prompts for confirmation at each step),
`on_table` (plate on table vs. wheelchair tray), `bite_transfer_height` (`TALL` / `SHORT` / `CUSTOM`).
