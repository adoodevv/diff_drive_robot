# Mine Rescue Rover: What Was Built and How to Run the Final Demo

This file covers two things:

1. **What was done**: every change made to the original `diff_drive_robot`
   package to turn it into a mine rescue rover simulation.
2. **How to run it**: the exact commands for the full simulation you show
   the judges, plus a demo script, fallbacks and troubleshooting.

For more detail on individual parts, see
[`mine_rescue_demo.md`](mine_rescue_demo.md) (demo and realism notes),
[`lora_bridge.md`](lora_bridge.md) (radio protocol) and
[`lora_simulation_testing.md`](lora_simulation_testing.md) (bridge testing).

---

## 1. The idea in one paragraph

A rover goes into a collapsed underground mine where GPS and Wi-Fi do not
work. It maps the tunnels with lidar SLAM, finds trapped miners with a
thermal camera, measures dangerous gases, and sends all of it back to a
command post on the surface over a long-range, low-bandwidth **LoRa** radio.
When the signal gets weak deep in the mine, the rover **drops relay radios**
behind it to extend the link. The operator console at the command post shows
**only what actually came through the radio**, so dropped packets, slow video
and a map that fills in gradually are all real effects of the link, not
decorations.

```
 GAZEBO (the mine + rover sensors)
   lidar, IMU, wheel odometry, RGB camera, thermal camera, headlights, beacon
        │
        ▼
 ROVER (onboard computer)
   EKF → SLAM Toolbox → map + pose
   mine_sensors       CH4 / CO / O2 / H2S / temperature / humidity / battery
   survivor_detector  thermal blobs + lidar → survivor position on the map
   autopilot          autonomous survey route, obstacle avoidance
   rover_radio        LoRa scheduler (alerts > telemetry > gas > map > video)
        │  raw LoRa packets (≤ 200 bytes each)
        ▼
 THE AIR: rf_channel
   link budget, tunnel/corner/rock-fall losses, fading → packet lost or
   delivered after its real airtime; relay drops when the margin gets thin
        │  surviving packets
        ▼
 COMMAND POST
   lora_receiver      the same node used with the real radio hardware
   operator console   http://localhost:8080 (map, video, thermal, gas, link,
                      alerts, drive / AUTO / HOLD / E-STOP / drop relay)
```

---

## 2. What was done

### 2.1 A generated underground mine world

* **`tools/mine_world/`** (new): a procedural generator (`generate.py`,
  `layout.py`, `geometry.py`, `textures.py`, `world.py`) that builds the whole
  mine from one layout description:
  * portal cut into a pit highwall, timbered haulage drift, crosscuts,
    room-and-pillar stope, a **rock-fall**, **flooded old workings**, a
    **refuge chamber**
  * **3 trapped miners** (warm bodies the thermal camera can see), **gas
    pockets**, mine lamps, dust, rails, ventilation duct, pipes, cables,
    safety signs, carts, containers
* **`models/mine_rescue/`** (new): the generated meshes (walls, floor,
  ceiling, rubble, rails, pipes, ...) and PBR textures (albedo / normal /
  roughness) plus warning-sign textures.
* **`worlds/mine_rescue.sdf`** (new): the Gazebo world that loads the mine.
  `worlds/obstacles.world` was also updated to the mine setting.
* **`config/mine_layout.yaml`** and **`config/mine_free_space.png`** (new):
  written by the generator, so the gas model, radio model and autopilot all
  use exactly the same geometry that Gazebo draws.
* **`config/gazebo_gui.config`** (new): Gazebo window layout with the
  cut-away view of the mine and a docked onboard-camera panel.

### 2.2 Rover hardware in simulation

* **`urdf/robot.urdf`, `urdf/robot.gazebo`** (modified): added a sensor
  head with
  * a **640×480 RGB camera**
  * a **160×120 radiometric thermal camera** (like a FLIR Lepton; outputs
    kelvin × 100)
  * two spot **headlights**, an amber **beacon**, and a LoRa whip antenna
* **`config/gz_bridge.yaml`** (modified): bridges the new camera and thermal
  topics, and separates **wheel odometry** (`wheel/odom`) from Gazebo's
  ground-truth `odom`.
* **`config/ekf.yaml`** (modified): the EKF now fuses real wheel odometry
  instead of ground truth, and uses only IMU orientation and yaw rate. Fusing
  linear acceleration turned tiny sensor tilt into tens of metres of drift.
* **`launch/rsp.launch.py`** (modified): wraps the xacro output in
  `ParameterValue(..., value_type=str)`, which newer ROS 2 releases need.

### 2.3 Rover software (new package module `rescue_sim/`)

| Node | What it does |
|---|---|
| `mine_sensors` | Simulated multi-gas head: CH4 (%LEL), CO, O2, H2S, temperature, humidity and battery. Uses realistic sensor lag (fast catalytic CH4, slow electrochemical CO) and industry alarm levels (20 %LEL CH4, 30 ppm CO, 19.5 % O2). Raises gas alerts. |
| `survivor_detector` | Finds people in the thermal image (30-36 °C against 20-24 °C rock), rejects hot lamps and engines, estimates range from the floor contact point and the lidar, and places the survivor on the SLAM map. Needs repeated sightings before raising an alert. Does **not** use simulator ground truth. |
| `autopilot` | Autonomous survey: pure-pursuit along the route in the layout, slows for obstacles, keeps off walls, backs out if stuck, pauses at survivors to sweep the thermal camera. |
| `rover_radio` | Onboard LoRa scheduler. Sends alerts first, then telemetry (2 Hz) and gas (1 Hz), and splits the remaining airtime between map tiles, camera JPEG slices and thermal slices. Never sends faster than the air can carry. Also owns `/cmd_vel` and arbitrates the modes MANUAL / AUTO / HOLD / E-STOP, with a 1 s deadman for remote driving. |
| `rf_channel` | The radio channel between rover and command post. Computes a link budget for the chosen radio profile, with tunnel waveguide loss, corner loss, rock-fall obstruction, shadowing and Rician fading, then drops or delivers each packet after its real airtime. **Automatically drops relay radios** when the margin gets thin, and spawns a relay model in Gazebo. |
| `operator_dashboard` | Serves the web operator console (`web/index.html`, `app.js`, `style.css`) at `http://localhost:8080`. It reads **only** `/lora/*` topics, meaning only what came through the radio. |
| `mine_effects` | Visual effects only: flickering failing lamps, the flashing collapse strobe, and blinking relay LEDs. |

Supporting libraries: `rescue_sim/lora_phy.py` (Semtech LoRa airtime,
sensitivity and packet error rate) and `rescue_sim/mine_map.py` (the mine as
the ROS side sees it, plus the tunnel propagation model).

### 2.4 LoRa protocol extended for rescue data

* **`lora_bridge/protocol.py`** (modified): new packet types, each fitting in
  a single LoRa packet (≤ 200 B):

  | ID | Direction | Content |
  |---|---|---|
  | `0x01` telemetry | rover → base | pose and velocity (existing; the ESP32 firmware sends this too) |
  | `0x02` environment | rover → base | gas, temperature, humidity, battery, mode flags (20 B) |
  | `0x03` map tile | rover → base | 24×24 cells, 2 bits each, zlib-compressed (typically 20-60 B) |
  | `0x04` image chunk | rover → base | a slice of a camera or thermal JPEG |
  | `0x05` alert | rover → base | survivor / gas / relay, with map position and confidence |
  | `0x10` command | base → rover | mode, drop-relay action, velocity |

* **`lora_bridge/map_assembler.py`** (new): rebuilds the occupancy map at the
  base station from tiles that arrive out of order or not at all.
* **`lora_bridge/lora_receiver_node.py`** (modified): decodes all new packet
  types and publishes `/lora/environment`, `/lora/map`,
  `/lora/camera/image/compressed`, `/lora/thermal/image/compressed` and
  `/lora/alerts`. It sends operator commands on the downlink at 5 Hz, which
  also serves as the rover's deadman heartbeat. It can use ROS topics instead
  of a serial port, which is how the simulated channel plugs in. **This is the
  same node that runs with the real radios.**
* **`lora_bridge/sim_rover_tx_node.py`**: replaced deprecated
  `logger.warn` with `logger.warning`.

### 2.5 New messages (`msg/`)

* `MineEnvironment.msg`: gas and housekeeping readings
* `RescueAlert.msg`: survivor / gas / relay-deployed alerts with position
* `OperatorCommand.msg`: HOLD / MANUAL / AUTO / E-STOP, drop relay, velocity
* `LinkStatus.msg`: RSSI, SNR, margin, packet error rate, hops, throughput,
  relay positions

### 2.6 Launch, config, visualisation

* **`launch/mine_rescue.launch.py`** (new): one command starts everything:
  Gazebo server and GUI, robot spawn, ROS-Gazebo bridge, EKF, SLAM Toolbox,
  all rover nodes, the RF channel, the receiver, the dashboard, effects, the
  browser and (optionally) RViz.
* **`config/lora_link.yaml`** (new): radio profiles and propagation/relay
  parameters.
* **`config/slam_toolbox_mine.yaml`** (new): SLAM settings tuned for tunnels.
* **`rviz/mine_rescue.rviz`** (new): engineering view comparing the rover's
  SLAM map with the map received over LoRa.
* **`CMakeLists.txt`, `package.xml`** (modified): new messages, install
  rules for `rescue_sim`, `models` and `web`, and the new dependencies.
* **`README.md`, `docs/lora_bridge.md`, `docs/mine_rescue_demo.md`**: updated
  and new documentation.

### 2.7 Tests

* **`test/test_rescue_link.py`** (new): round-trip tests for every new
  payload, map and image reassembly, LoRa airtime against Semtech's
  calculator, and the tunnel propagation model.
* The full test suite currently passes: **46 passed**.

---

## 3. One-time setup

Run these once on the demo machine (they are safe to re-run).

```bash
# 1. ROS 2 environment (this machine has ROS 2 Lyrical; use jazzy if that is
#    what's installed)
source /opt/ros/lyrical/setup.bash

# 2. Install all dependencies listed in package.xml
#    (slam_toolbox, robot_localization, ros_gz, xacro, numpy, opencv, ...)
cd ~/ros2_ws
rosdep install --from-paths src --ignore-src -r -y

# 3. Build the package
colcon build --packages-select diff_drive_robot --symlink-install

# 4. Sanity check: the unit tests (should print "46 passed")
cd ~/ros2_ws/src/diff_drive_robot
python3 -m pytest -q test/
```

> If you ever edit the mine layout, regenerate the world **before** building:
> `python3 tools/mine_world/generate.py` (about 20 s, needs numpy and OpenCV).

---

## 4. Running the final demo

### 4.1 The main command (this is what you show the judges)

Open **one terminal**:

```bash
source /opt/ros/lyrical/setup.bash
cd ~/ros2_ws
source install/setup.bash

ros2 launch diff_drive_robot mine_rescue.launch.py autopilot:=true rviz:=true
```

What opens:

* **Gazebo**: a cut-away view of the whole mine with the rover at the portal,
  and a docked panel showing the rover's onboard camera at full rate.
* **Operator console** in your browser at **http://localhost:8080** (opens by
  itself after about 6 s; open it manually if not).
* **RViz** (because of `rviz:=true`): an engineering view comparing the
  rover's own SLAM map with the map received over LoRa. Leave out
  `rviz:=true` if the laptop is slow.

With `autopilot:=true` the rover starts its survey by itself. Without it, the
rover starts in MANUAL and you click **Autonomous survey** on the console,
which is a nicer moment to show live because the command goes over the LoRa
downlink.

**Recommended for judges:** start without the autopilot flag and hand over
control on stage:

```bash
ros2 launch diff_drive_robot mine_rescue.launch.py rviz:=true
```

To stop everything: `Ctrl+C` in that terminal.

### 4.2 Optional extra terminals during the demo

Each new terminal first needs:

```bash
source /opt/ros/lyrical/setup.bash && source ~/ros2_ws/install/setup.bash
```

Then any of:

```bash
# Drive from the keyboard locally (works in MANUAL mode when the console is idle)
ros2 run teleop_twist_keyboard teleop_twist_keyboard

# Watch what arrives at the command post over the radio
ros2 topic echo /lora/alerts            # survivor / gas / relay alerts
ros2 topic echo /lora/environment       # gas readings as received
ros2 topic hz /lora/camera/image/compressed   # real video frame rate over LoRa

# See everything that is running
ros2 node list
ros2 topic list | grep lora
```

### 4.3 Demo script (about 10 minutes)

1. **Start** with the command in 4.1. Explain the three parts: the rover (in
   Gazebo), the air (radio model), and the command post (browser). Point out
   that the console shows only what survived the radio.
2. **Hand over to the robot**: click **Autonomous survey**. The mode pill
   changes only after the rover acknowledges over LoRa.
3. **Old workings** (first ~90 s): the map builds tile by tile. Methane
   rises, the route trace turns yellow and red, and the **gas alarm** banner
   and beeps fire. The rover passes under the hazard tape and finds
   **survivor 1** by the flooded sump, shown as a red marker with body
   temperature.
4. **Into the stope**: turning into crosscut B, the radio margin collapses
   (watch the link chart). The rover **drops a LoRa relay**: a yellow box
   with a blinking green LED appears in Gazebo, the console shows
   `GATEWAY ◀ R1 ◀ ROVER`, and the link recovers.
5. **Survivor 2** in the room-and-pillar stope (a second relay goes down),
   then deeper to the **refuge chamber** and **survivor 3** via a third relay.
   Point out that throughput and video frame rate drop with every extra hop,
   as they would with real relays.
6. **Rock-fall**: the rover threads the gap beside the rubble under the
   flashing red beacon, with CO and heat rising.
7. **Take control**: on the console, **W/A/S/D** or arrow keys drive over
   LoRa (1 s deadman: let go and it stops), **Space** = HOLD, **Esc** =
   E-STOP. The **Drop relay** button deploys a relay manually.

Useful Gazebo tricks: right-click the rover → **Follow** for a chase camera.

### 4.4 Showing the radio trade-off (optional second run)

Restart with a long-range, low-speed radio setting:

```bash
ros2 launch diff_drive_robot mine_rescue.launch.py link_profile:=e22_5k5
```

This gives more range per hop, but video drops to a frame every few seconds.
This shows the judges that the system respects real LoRa physics.

Available profiles (from `config/lora_link.yaml`):

| Profile | Radio | Air rate |
|---|---|---|
| `e22_62k5` (default) | Ebyte E22-900T22D, SF5 / 500 kHz | 62.5 kbps |
| `e22_21k9` | E22-900T22D, SF7 / 500 kHz | 21.9 kbps |
| `e22_5k5` | E22-900T22D, SF7 / 125 kHz | 5.5 kbps |
| `sx1280_2g4` | SX1280 2.4 GHz, SF6 / 1.6 MHz | 203 kbps |

### 4.5 All launch options

| Argument | Default | Meaning |
|---|---|---|
| `autopilot` | `false` | start in autonomous survey instead of manual |
| `link_profile` | `e22_62k5` | radio profile (table above) |
| `gui` | `true` | show the Gazebo window |
| `rviz` | `false` | open the RViz engineering view |
| `dashboard` | `true` | serve the operator console |
| `dashboard_port` | `8080` | console port |
| `open_browser` | `true` | open the console automatically |
| `effects` | `true` | failing lamps, collapse strobe, relay LEDs |
| `gz_verbosity` | `0` | Gazebo log level 0-4 (raise when debugging) |
| `extra_gz_args` | `''` | extra `gz sim` server flags, e.g. `--headless-rendering` |

---

## 5. Points worth saying to the judges

What is physically grounded:

* **LoRa airtime** uses Semtech's formula (checked against Semtech's
  calculator). Frame rates and map latency come from real airtime.
* **Packet loss** follows the receiver's sensitivity and SNR floor per
  spreading factor, with shadowing and fading.
* **Underground propagation**: tunnel waveguide loss per metre, extra loss at
  each corner, and obstruction loss through the rock-fall. The values are
  mid-range figures from published mine measurements.
* **Thermal detection** works on a radiometric image. Lamps and engines are
  hotter than a person and are rejected. No ground truth is used.
* **Gas sensors** have realistic response lag and standard alarm levels.
* The **command-post receiver is the same code** that runs with the real
  radio hardware, so the simulation tests the real software path.

Honest limits (saying these builds credibility):

* Sub-GHz LoRa cannot carry real-time video. The feed is **slow-scan JPEG**,
  about 2 fps at 62.5 kbps and much less at long-range settings.
* The E22's fastest air rate needs its UART raised above the 9600 baud
  default, otherwise the serial port becomes the bottleneck.
* The mine is procedurally generated; the geometry and textures are
  synthetic.

---

## 6. Troubleshooting on demo day

| Problem | Fix |
|---|---|
| `Package 'diff_drive_robot' not found` | You forgot `source ~/ros2_ws/install/setup.bash` |
| Gazebo looks black or very dark | The mine is unlit except for its lamps. Orbit down into a drift, or right-click the rover → **Follow**. |
| Console shows **LINK LOST** at start | Normal for a few seconds while Gazebo loads and SLAM starts |
| Browser didn't open | Open **http://localhost:8080** manually |
| Port 8080 is busy | Add `dashboard_port:=8090` and open http://localhost:8090 |
| Simulation is slow or laggy | Add `effects:=false`, leave out `rviz:=true`, and plug in the charger. A discrete GPU helps a lot. |
| Leftover processes from a previous run | `pkill -f "gz sim"; pkill -f diff_drive_robot` then launch again |
| Something failed and you need details | Relaunch with `gz_verbosity:=3` |

**Before presenting:** do one full dry run on the same laptop, with the
charger connected, and close other heavy applications.
