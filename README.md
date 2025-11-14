# Simulated Robots Package

Simulation for differential drive robots using ROS2 Jazzy and Gazebo Harmonic. This package provides all of the necessary files to get a simulated robot up and running. This includes the urdf, parameters and launch files for a robot with a lidar sensor and tele-operated navigation. More sensors and functionalities will be added in future.

## Branches

This repository has two main branches:

- **`main`**: Base repository with core robot simulation functionality including URDF, Gazebo simulation, and basic tele-operation
- **`mapping`**: Extended branch that adds mapping capabilities using SLAM Toolbox and robot localization with Extended Kalman Filter (EKF)

## Work in progress

The package is still being worked on and in development

## Supported on

Supported for [Ubuntu 24.04](https://releases.ubuntu.com/noble/) & [ROS2 Jazzy](https://docs.ros.org/en/jazzy/Installation.html) but compatibility with other versions has not been checked.

## Install Required ROS 2 Packages

### Base Packages (Required for all branches)

Make sure to install the following ROS 2 Jazzy Packages:

```bash
sudo apt install -y                         \
   ros-jazzy-ros-gz                        \
   ros-jazzy-ros-gz-bridge                 \
   ros-jazzy-joint-state-publisher         \
   ros-jazzy-xacro                         \
   ros-jazzy-teleop-twist-keyboard         \
   ros-jazzy-teleop-twist-joy 
```

### Additional Packages (Required for `mapping` branch)

If you're using the `mapping` branch, also install:

```bash
sudo apt install -y                         \
   ros-jazzy-slam-toolbox                  \
   ros-jazzy-robot-localization
```

## Install

To use this package please download all of the necessary dependencies first and then follow these steps:

### Main Branch

For the base robot simulation, clone the main branch:

```bash
mkdir -p ros2_ws/src
cd ros2_ws/src
git clone https://github.com/adoodevv/diff_drive_robot.git
cd ..
colcon build --packages-select diff_drive_robot --symlink-install
```

### Mapping Branch

To use the mapping features, clone the `mapping` branch directly:

```bash
mkdir -p ros2_ws/src
cd ros2_ws/src
git clone -b mapping https://github.com/adoodevv/diff_drive_robot.git
cd ..
colcon build --packages-select diff_drive_robot --symlink-install
```

## Usage

### Main Branch - Basic Robot Simulation

After sourcing ROS and this package, launch the 2-wheeled differential drive robot simulation:

```bash
source install/setup.bash
ros2 launch diff_drive_robot robot.launch.py 
```

#### Controlling the robot

Currently, only keyboard control works. Run this in another terminal:

```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard 
```

![Gazebo Simulation](assets/gazebo_mapping.png)

### Mapping Branch - Mapping and Localization

The `mapping` branch extends the base functionality with:

- **SLAM Toolbox**: For mapping the environment
- **Extended Kalman Filter (EKF)**: For sensor fusion and improved odometry estimation
- **Enhanced RViz configuration**: Pre-configured for mapping visualization

#### New Configuration Files

The mapping branch includes additional configuration files:

- `config/slam_toolbox_mapping.yaml`: Configuration for SLAM Toolbox mapping mode
- `config/ekf.yaml`: Extended Kalman Filter parameters for sensor fusion (odometry and IMU)

#### Launching the Mapping System

The mapping system requires launching two separate terminals:

**Terminal 1 - Robot Simulation:**
```bash
source install/setup.bash
ros2 launch diff_drive_robot robot.launch.py
```

This launches:
- Gazebo simulation
- Robot State Publisher
- Gazebo-ROS bridge
- Extended Kalman Filter node
- Robot spawner

**Terminal 2 - Mapping:**
```bash
source install/setup.bash
ros2 launch diff_drive_robot mapping.launch.py
```

This launches:
- SLAM Toolbox (online async mapping mode)
- RViz with mapping configuration

#### Controlling the robot during mapping

In a third terminal, run the keyboard teleop:

```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

![RViz Mapping](assets/rviz_mapping.png)

#### Saving the Map

Once you've mapped your environment, you can save it using the SLAM Toolbox plugin in RViz or via command line:

```bash
ros2 run nav2_map_server map_saver_cli -f ~/my_map
```

## TODO

Package is still being worked on, though the core functionality is pretty much done, I will be adding some more sensors and functionalities soon.
