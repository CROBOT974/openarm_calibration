# OpenArm Calibration

Hand-eye calibration package for OpenArm using the easy_handeye2 library (eye-on-base mode).

## Prerequisites (one-time)

```bash
# Clone easy_handeye2
cd ~/openarm/ros2_ws/src/openarm_ros2
vcs import < openarm.repos

# Build
cd ~/openarm/ros2_ws
source /opt/ros/humble/setup.bash
colcon build --symlink-install
```

## Per-terminal Setup

Source these in every new terminal before running any commands:

```bash
source /opt/ros/humble/setup.bash
source ~/openarm/ros2_ws/install/setup.bash
```

## 1. Camera Intrinsic Calibration (one-time)

Place a chessboard in front of the camera and show it from multiple angles.

```bash
cd ~/openarm/ros2_ws/src/openarm_ros2/openarm_calibration/scripts
python3 camera_calib.py   # images saved under Camera/test2
```

- Press `c` to capture frames (collect 15-30 frames).
- Press `s` to compute and display calibration results.
- Copy the output `fx`, `fy`, `cx`, `cy` values into the `DEFAULT_CAMERA_MATRIX` constant in `aruco_pose_node.py`.

## 2. Hand-Eye Calibration

### Terminal 1 — Launch robot, camera, and easy_handeye2

```bash
ros2 launch openarm_calibration calibrate.launch.py
```

### Terminal 2 — Auto-sampling

Moves the arm through predefined poses. After each pose, click **Take Sample** in the rqt panel, then press Enter to continue.

```bash
ros2 run openarm_calibration auto_sample.py \
  ~/openarm/ros2_ws/src/openarm_ros2/openarm_calibration/scripts/calibration_poses.yaml
```

### Terminal 3 — ArUco marker detection

```bash
ros2 run openarm_calibration aruco_pose_node.py \
  --camera-frame tr_base \
  --marker-frame tr_marker
```

After collecting **15-20 samples**, click **Compute Calibration** in the rqt panel.

The result is saved to `~/.ros2/easy_handeye2/calibrations/openarm_right_eob.calib`.

## 3. Verify Calibration Result

### Terminal 1 — Simulated robot (no physical robot needed)

```bash
ros2 launch openarm_bringup openarm.bimanual.launch.py \
  arm_type:=v10 use_fake_hardware:=true
```

### Terminal 2 — ArUco marker detection

```bash
ros2 run openarm_calibration aruco_pose_node.py \
  --camera-frame tr_base \
  --marker-frame tr_marker
```

### Terminal 3 — Publish calibration TF

```bash
ros2 run openarm_calibration publish_calib.py \
  --name openarm_right_eob
```

### Terminal 4 — Move arm and verify marker alignment

```bash
ros2 action send_goal /right_joint_trajectory_controller/follow_joint_trajectory \
  control_msgs/action/FollowJointTrajectory \
  '{trajectory: {joint_names: ["openarm_right_joint1","openarm_right_joint2","openarm_right_joint3","openarm_right_joint4","openarm_right_joint5","openarm_right_joint6","openarm_right_joint7"], points: [{positions: [0.3, 0.0, 0.0, 0.5, 0.0, 0.0, 0.0], time_from_start: {sec: 3, nanosec: 0}}]}}'
```

Check that `tr_marker` is aligned with the robot end-effector in RViz.

## 4. Daily Usage (calibration result already on robot)

```bash
# Launch robot + publish calibration TF
ros2 launch openarm_bringup openarm.bimanual.launch.py \
  arm_type:=v10 use_fake_hardware:=false

ros2 run openarm_calibration publish_calib.py --name openarm_right_eob
```

## TF Frame Graph

During calibration:

```
world ──→ openarm_right_hand   (robot kinematics)
 tr_base ──→ tr_marker          (ArUco detection)
```

After publishing calibration result:

```
world ──→ tr_base               (calibration static TF)
 tr_base ──→ tr_marker           (ArUco detection)
```

## Nodes

| Node | Description |
|------|-------------|
| `aruco_pose_node.py` | Detects ArUco marker via Hikrobot USB3 camera, publishes `tr_base → tr_marker` TF |
| `auto_sample.py` | Moves the right arm through predefined poses for calibration sampling |
| `publish_calib.py` | Reads `.calib` result and publishes `world → tr_base` as a static TF |

## License

[Apache License 2.0](LICENSE)

Copyright 2025 Enactic, Inc.
