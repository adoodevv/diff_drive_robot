# Simulated Robots Package

Simulation for differential drive robots using ROS2 Jazzy and Gazebo Harmonic. This package provides all of the necesarry files to get a simulated robot up and running. This includes the urdf, parameters and launch files for a robot with a lidar sensor and tele-operated navigation. More sensors will be added in future.

## Work in progress

The package is still being worked on and in development

## Supported on

Supported for [Ubuntu 24.04](https://releases.ubuntu.com/noble/) & [ROS2 Jazzy](https://docs.ros.org/en/jazzy/Installation.html) but compatibility with other versions has not been checked.

## Usage

## Install Required ROS 2 Packages

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

## Install

To use this package please download all of the necessary dependencies first and then follow these steps

```bash
mkdir -p ros2_ws/src
cd ros2_ws/src
git clone https://github.com/adoodevv/diff_drive_robot.git
cd ..
colcon build --packages-select diff_drive_robot --symlink-install
```

### Launch Gazebo simulation together with Rviz

After sourcing ROS and this package we can launch the 2-wheeled differential drive robot simulation with the following command:

```bash
source install/setup.bash
ros2 launch diff_drive_robot robot.launch.py 
```

### Controlling the robot

Currently, only keyboard control works. Run this in another terminal:

```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard 
```

## TODO

Package is still being worked on, though the core funtionality is pretty much done, I will be adding some more sensors and functionalities soon.
