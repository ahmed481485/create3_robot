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

**Terminal 1 – start the simulation**

```bash
cd <path-to>/create3_ws
source install/setup.bash
export LIBGL_ALWAYS_SOFTWARE=1
ros2 launch irobot_create_gz_bringup create3_gz.launch.py world:=maze visualize_rays:=false
```

** Start diferent**

colcon build --packages-select irobot_create_gz_bringup && source install setup.bash
ros2 launch irobot_create_gz_bringup create3_gz.launch.py world:=arena


Then press ▶ in Gazebo.

**Terminal 2 – activate the controllers**

```bash
source install/setup.bash
ros2 control switch_controllers --activate joint_state_broadcaster diffdrive_controller
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

## Real robot with RViz

**Check the connection**

```bash
ros2 topic list

# Check ROS settings
echo $ROS_DOMAIN_ID
echo $RMW_IMPLEMENTATION
```

**Terminal 1 – drive**

```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

**Terminal 2 – robot model (optional)**

```bash
ros2 launch irobot_create_common_bringup robot_description.launch.py use_sim_time:=false
```

**Terminal 3 – RViz**

```bash
rviz2
```
