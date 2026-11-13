# ADVRF ROS 2 Workspace

ROS 2 workspace for Miranda robot control using the ADVRF EtherCAT framework. It supports development with a dummy EtherCAT master and operation on the real robot.

Tested with **Ubuntu 24.04** and **ROS 2 Jazzy**.

## Prerequisites

### 1. Create the workspace directory

Keep the EtherCAT workspace, robot configuration, and ROS 2 workspace under a common directory:

```bash
mkdir -p ~/ws
```

The final workspace should look like that:

| Path | Contents |
| --- | --- |
| `~/ws/code_ws` | EtherCAT master/client workspace |
| `~/ws/configs/ecat-miranda-config` | Miranda EtherCAT configuration |
| `~/ws/ros2_ws` | This ROS 2 workspace |

### 2. Install ROS 2 Jazzy

Follow the [official Ubuntu installation instructions](https://docs.ros.org/en/jazzy/Installation/Ubuntu-Install-Debs.html).

### 3. Install the ADVRF EtherCAT framework

```bash
cd ~/ws
git clone https://github.com/Advanced-Robotics-Facility/xenomai-ethercat-setup.git
cd xenomai-ethercat-setup
./build_ecat.sh
```

When prompted:

- Set the workspace name to `code_ws`.
- Set the workspace parent directory to the absolute path of `~/ws`, such as `/home/your-user/ws`.
- If you are installing the framework on your personal computer:
  - Choose No to `Enable Xenomai Real-Time?`
  - Choose Yes to  `Build GUI Tools?`
- Select all packages and dependencies.

Access to the private `ecat-master-advr` repository is required to build the master from source.

### 4. Clone the robot configuration

```bash
mkdir -p ~/ws/configs
cd ~/ws/configs
git clone --branch ec_ros2 https://github.com/Advanced-Robotics-Facility/ecat-miranda-config.git
```

## Installation and setup

### 1. Clone this workspace

Clone the repository with its submodules:

```bash
cd ~/ws
git clone --recurse-submodules --branch jazzy_ec/clean https://github.com/Advanced-Robotics-Facility/ws_advrf_ros2.git ros2_ws
cd ~/ws/ros2_ws
```

If you already cloned the repository without submodules, initialize them with:

```bash
git submodule update --init --recursive
```

### 2. Configure the build

Edit `colcon.meta` and update the following CMake arguments:

| Argument | Value |
| --- | --- |
| `-DECAT_INSTALL_PREFIX` | `/home/your-user/ws/code_ws/install` |
| `-DECAT_CLIENT_SOURCE_DIR` | `/home/your-user/ws/code_ws/src/ecat-client-advr` |
| `-DXENOMAI` | `OFF` on your personal computer; `ON` on the robot embedded PC |

### 3. Load the environment

Run the following in every terminal used to build or launch the system:

```bash
source /opt/ros/jazzy/setup.bash
source ~/ws/code_ws/setup.bash
source ~/ws/configs/ecat-miranda-config/setup.sh

# Replace `0` if your setup uses a different ROS domain ID.
# Use the same domain ID on your PC and the robot.
export ROS_DOMAIN_ID=0
```

### 4. Install dependencies and build

Initialize rosdep once per machine. Skip this command if it is already initialized:

```bash
sudo rosdep init
```

Install the dependencies and build:

```bash
cd ~/ws/ros2_ws

rosdep update
rosdep install --from-paths src --ignore-src --rosdistro jazzy -r -y

sudo apt install -y ros-jazzy-gz-sim-vendor

colcon build --symlink-install
```

**Note1:** If the build fails because CMake cannot find `gz-sim8` or its dependencies, follow the [Gazebo dependency troubleshooting](#gazebo-dependency-troubleshooting) instructions below.

**Note2:** If the build fails because CMake cannot find `xbot2_interface`, follow the [Gravity Compensation Dependencies Installation](#gravity-compensation-dependencies-installation) instructions below. Otherwise, if you don't need the Gravity Compensation example just add a COLCON_IGNORE file inside src/advrf_ros2_gravity_compensation folder. 

After a successful build, source the workspace:

```bash
source ~/ws/ros2_ws/install/setup.bash
```

Run this command after the environment commands above in every new terminal used to launch workspace packages.

## Running with the dummy EtherCAT master

The dummy master simulates EtherCAT devices for development without the real robot.

### 1. Configure the dummy devices

Edit the master configuration files:

```bash
vim ~/.ecat_master/configs/nrt_config.yaml
vim ~/.ecat_master/configs/ecat_dummy_cfg.yaml
```

In `nrt_config.yaml`, set `eth_iface` to your network interface. To list available interfaces:

```bash
ip link show
```

In `ecat_dummy_cfg.yaml`, configure the devices and EtherCAT IDs to match the robot.

For the Miranda arm, replace the `SYNAPTICON_v301: 10` entry under the existing device mapping with:

```yaml
CENTAC_v15: [11, 12, 13, 14, 15, 16, 302]
```

Preserve the surrounding YAML structure and indentation. This simulates the seven arm motors with the expected EtherCAT IDs.

### 2. Start the dummy master

In a terminal with the environment loaded:

```bash
repl -d
```

### 3. Launch a control interface

In a second terminal, load the environment and source the ROS 2 workspace. Then choose one of the following examples.

#### Slider UI or joystick control

```bash
ros2 launch advrf_ros2_miranda_bringup miranda.launch.py \
  start_bridge:=true \
  use_joy:=true \
  active_tx_on_first_msg:=true \
  control_mode:=position
```

#### MoveIt 2 planning and execution

```bash
ros2 launch advrf_ros2_miranda_bringup moveit.launch.py \
  start_bridge:=true \
  active_tx_on_first_msg:=true \
  control_mode:=impedance
```

See the individual package READMEs for launch arguments, controller configuration, and usage details.

## Running on the real robot

Run the EtherCAT master and ROS 2 bridge on the robot's embedded PC. Run the control interface on your personal computer.

Both machines must use the same `ROS_DOMAIN_ID` and be able to communicate over the network. Load the environment and source the ROS 2 workspace in each terminal.

### 1. On the robot embedded PC

Start the EtherCAT master:

```bash
repl
```

In a separate terminal, start the ROS 2 bridge:

```bash
ros2 launch advrf_ros2_miranda_bringup bridge.launch.py \
  active_tx_on_first_msg:=true
```

### 2. On your personal computer

Choose one of the following control interfaces.

Disable the local bridge because it is already running on the embedded PC.

#### Slider UI or joystick control

```bash
ros2 launch advrf_ros2_miranda_bringup miranda.launch.py \
  use_joy:=true \
  control_mode:=position
```

#### MoveIt2 planning and execution

```bash
ros2 launch advrf_ros2_miranda_bringup moveit.launch.py \
  control_mode:=impedance
```

#### Gravity Compensation example

```bash
ros2 launch advrf_ros2_miranda_bringup gravity_compensation.launch.py \
  use_rviz:=true
```

## Viewing data with PlotJuggler

Install PlotJuggler:

```bash
sudo apt install -y ros-jazzy-plotjuggler-ros
```

With the ROS environment loaded, launch it with:

```bash
ros2 run plotjuggler plotjuggler
```

# Troubleshooting

## Gazebo dependency troubleshooting

If `dagana_gazebo` fails to build because CMake cannot find `gz-sim8` or dependencies such as `sdformat14`, install the system Gazebo Harmonic development package.

Add the official Gazebo apt repository and install the development files:

```bash
sudo apt update
sudo apt install -y curl lsb-release gnupg

sudo curl -fsSL https://packages.osrfoundation.org/gazebo.gpg \
  -o /usr/share/keyrings/pkgs-osrf-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/pkgs-osrf-archive-keyring.gpg] https://packages.osrfoundation.org/gazebo/ubuntu-stable $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/gazebo-stable.list > /dev/null

sudo apt update
sudo apt install -y libgz-sim8-dev
```

If you previously added manual Gazebo path overrides, open a fresh terminal.

Load the environment described above, then clear the CMake cache and rebuild:

```bash
cd ~/ws/ros2_ws
colcon build --symlink-install --cmake-clean-cache
source install/setup.bash
```

## Gravity Compensation Dependencies Installation

> Momentarily the Gravity Compensation package depends on HHCM Research Line xbot2_interfaces API, the idea is to directly implement a Pinocchio backend ourselves.

```bash
sudo apt-get update && sudo apt-get install -y wget gnupg curl gnupg2 ca-certificates lsb-release

sudo sh -c 'echo "deb https://advrhumanoids.github.io/XBotBuild/ubuntu/noble ./" > /etc/apt/sources.list.d/xbot.list'
curl -fsSL https://advrhumanoids.github.io/XBotBuild/KEY.gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/xbot2.gpg

sudo apt update
sudo apt install xbot_env xbot2_interface

source /opt/xbot/setup.sh
```

Load the environment described above, then clear the CMake cache and rebuild:

```bash
cd ~/ws/ros2_ws
colcon build --symlink-install --cmake-clean-cache
source install/setup.bash
```
