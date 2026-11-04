# create3_ws

## Setup 

Requires ROS 2 Jazzy.

```bash
git clone <repo-url> create3_ws
cd create3_ws
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install
```

## Run the simulation


Then press ▶ in Gazebo.

**Terminal 1 – activate the controllers**

```bash
source install/setup.bash
ros2 control switch_controllers --activate joint_state_broadcaster diffdrive_controller

```
**Terminal 2 – start the simulation**

```bash
cd <path-to>/create3_ws
source install/setup.bash
export LIBGL_ALWAYS_SOFTWARE=1
ros2 launch irobot_create_gz_bringup create3_gz.launch.py world:=arena visualize_rays:=false spawn_dock:=false
```

**Terminal 3 – drive with the keyboard** (`i` / `j` / `l` / `k`)

```bash
source install/setup.bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard --ros-args -p stamped:=true
```

**Stop / clean up**

```bash
pkill -f "gz sim"

# If the VM freezes (run on the Mac host)
pkill -9 vmware-vmx
```

## SLAM (slam_toolbox)

With the simulation running and the controllers active:

```bash
source install/setup.bash
ros2 launch slam_toolbox online_async_launch.py slam_params_file:=src/create3_sim/config/mapper_params_online_async.yaml use_sim_time:=true
```

- The argument is `slam_params_file`, not `params_file`. Unknown arguments are silently ignored.
- Use `use_sim_time:=true` in sim and `false` on the real robot.
- RViz: `rviz2 --ros-args -p use_sim_time:=true`, Fixed Frame `map`, add a Map display on `/map`.
- Save from RViz (Panels → SlamToolbox) with a full path like `/home/ahmed/maps/arena`. **Save Map** writes `.pgm`/`.yaml` for Nav2 and needs `ros-jazzy-nav2-map-server`. **Serialize Map** writes a pose graph that slam_toolbox can reload.

## SLAM on the real robot (Humble, untested)

Run everything on the robot side with Humble. Install `ros-humble-slam-toolbox` and `sllidar_ros2`.

```bash
# 1. Lidar driver (computer the C1 is plugged into)
ros2 launch sllidar_ros2 sllidar_c1_launch.py serial_port:=/dev/ttyUSB0 frame_id:=rplidar_scan_frame

# 2. Lidar position on the robot (measure the real height of the laser above the floor)
ros2 run tf2_ros static_transform_publisher --x 0 --y 0 --z 0.126 --frame-id base_link --child-frame-id rplidar_scan_frame

# 3. Check odometry (published by the Create 3 itself, no controllers to activate)
ros2 run tf2_ros tf2_echo odom base_link

# 4. SLAM: sim time must be false
ros2 launch slam_toolbox online_async_launch.py slam_params_file:=<path>/mapper_params_online_async.yaml use_sim_time:=false
```

- Serial "permission denied": `sudo usermod -aG dialout $USER`, then log in again.
- Sync the clocks of the companion computer and the Create 3 (NTP / `chrony`). Otherwise you get "Failed to compute odom pose" or TF extrapolation errors.
- `ROS_DOMAIN_ID` and `RMW_IMPLEMENTATION` must match the robot's web interface settings.
- RViz: plain `rviz2`, without sim time.

## Robot model on the real robot (Humble, untested)

Build only the three description packages (the stock apt version has no lidar):

| Package | Why |
|---|---|
| `irobot_create_description` | URDF/xacro with the lidar, meshes |
| `irobot_create_control` | the xacro looks up `$(find irobot_create_control)`, so it must be installed |
| `irobot_create_common_bringup` | `robot_description.launch.py`, RViz config |

```bash
git clone https://github.com/ahmed481485/create3_robot.git ~/create3_ws
cd ~/create3_ws
source /opt/ros/humble/setup.bash
sudo apt install ros-humble-xacro ros-humble-robot-state-publisher ros-humble-joint-state-publisher ros-humble-rviz2
colcon build --packages-select irobot_create_control irobot_create_description irobot_create_common_bringup \
  --packages-ignore irobot_create_nodes irobot_create_msgs   # bringup lists nodes as a dependency, but the model doesn't need it
source install/setup.bash

# Model + TF (base_link → all links, incl. rplidar_scan_frame)
ros2 launch irobot_create_common_bringup robot_description.launch.py gazebo:=ignition use_sim_time:=false
rviz2
```

- RViz: Fixed Frame `odom` (`map` if SLAM is running). Add a RobotModel display on `/robot_description`.
- This replaces the `static_transform_publisher` from the SLAM section. Run one or the other, not both.
- Set the measured lidar height in the URDF first (`<origin>` in `create3.urdf.xacro`, `scan_plane_z`).

## RPLIDAR C1 (simulation)

An RPLIDAR C1 sits on top of the robot, centred. Its scans are published on `/scan` (`sensor_msgs/LaserScan`).

### What was changed

| File | Change |
|---|---|
| `irobot_create_description/urdf/sensors/rplidar_c1.urdf.xacro` | **New.** Lidar macro: links, joints, `gpu_lidar` sensor (360°, 500 samples, 10 Hz, 0.05–12 m) |
| `irobot_create_description/urdf/create3.urdf.xacro` | Includes the macro, adds `body_top_z` (9.47 cm), calls `<xacro:rplidar_c1>` |
| `irobot_create_gz_bringup/launch/create3_ros_gz_bridge.launch.py` | Adds `lidar_bridge` (gz `/scan` → ROS `/scan`) |

Nothing else was needed. The existing `Sensors` plugin runs any `gpu_lidar`, `robot_state_publisher` publishes the new TF frames from the URDF, and the bridge launch is already included by the spawn launch.

### Frames

```
base_link (floor, centre of robot)
  └─ rplidar_body_joint  (+0.0947 m, set by <origin> in create3.urdf.xacro)
     rplidar_body          the lidar body (visual, collision, 110 g)
       └─ rplidar_joint  (+0.031 m, scan_plane_z, an estimate)
          rplidar_scan_frame   invisible point the laser measures from; frame_id of /scan
```

The scan plane is about 12.6 cm above the floor. To change where the lidar is mounted, edit the `<origin>` in the macro call. To change the laser height inside the lidar, edit `scan_plane_z`.

### Check and visualize

```bash
ros2 topic hz /scan                      # ~10 Hz × real-time factor (5 Hz at RTF 0.5 is normal)
ros2 topic echo /scan --once --field header
ros2 run tf2_ros tf2_echo base_link rplidar_scan_frame
```

RViz (in a sourced terminal, otherwise the meshes fail to load):

```bash
source install/setup.bash
rviz2 --ros-args -p use_sim_time:=true   # without sim time RViz drops the scans
```

Then set Fixed Frame to `base_link` and add a LaserScan display on `/scan`. In Gazebo, launch with `visualize_rays:=true` to see the rays.

On the real robot, run the Slamtec driver with `frame_id:=rplidar_scan_frame`.

### Xacro basics

- **Macro** = a template. `<xacro:macro name="rplidar_c1" params="...">` defines it, and `<xacro:rplidar_c1 .../>` uses it. The tag name must match `name`, and the file must be loaded with `<xacro:include>`.
- **Params**: `gazebo`, `namespace` have no default, so they're required. `parent:=base_link` has a default, so it can be left out of the call.
- **`*origin`** is a block param: the `<origin>` written *inside* the call is pasted where the macro has `<xacro:insert_block name="origin"/>` (here, into `rplidar_body_joint`).
- **`$(arg x)`** reads a top-level argument set by the launch file (e.g. `gazebo:=ignition`). **`${x}`** reads a param or property and can do math.
- **`xacro:inertial_dummy`** gives a tiny 10 g mass to a link that is only a frame. Gazebo drops links without `<inertial>`, and any sensor on them would disappear with the link.
- **`preserveFixedJoint`** stops gz from merging a fixed-joint link into its parent.

### Launch file basics

- `Node(...)` from `launch_ros.actions` only **describes** a program to start (`package` + `executable`, plus `parameters` / `remappings`). It starts when added with `ld.add_action(...)` and run with `ros2 launch`, the same as `ros2 run <package> <executable>`.
- `DeclareLaunchArgument` declares command-line options (`world:=arena`). `IncludeLaunchDescription` runs another launch file.

### Why the bridge is needed

Gazebo and ROS 2 are separate messaging systems (gz-transport vs. ROS/DDS), with different topics and message types (`gz topic -l` vs. `ros2 topic list`). A normal ROS node can't see Gazebo topics. `parameter_bridge` (package `ros_gz_bridge`) contains both a gz node and a ROS node. It subscribes on one side, converts the message, and publishes on the other:

```
/scan@sensor_msgs/msg/LaserScan[gz.msgs.LaserScan
topic  ROS type                 │ gz type
                                └ [ gz→ROS   ] ROS→gz   @ both
```
