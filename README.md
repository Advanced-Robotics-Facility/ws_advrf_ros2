```bash
source /opt/ros/jazzy/setup.bash
source ~/ros2_ws/install/setup.bash
source ~/path/to/miranda/config/setup.sh
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
export ROS_DOMAIN_ID=42

ros2 launch advrf_ros2_miranda_bringup moveit.launch.py use_advrf:=true
```