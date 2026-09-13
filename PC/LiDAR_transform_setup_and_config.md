# LiDAR Bringup — Ouster OS-1-32-U3 on the Dual-PC Stack

**Status:** working as of 2026-09-13
**Supersedes:** the Zenoh sections of `zenoh_dds_setup.md` and the IP addresses in `LAN_cable_setup.md` (see §10)

This document takes you from a cold start — laptop, hotspot, two unpowered machines — to a LiDAR publishing points with a correct TF tree. It also records every trap we hit getting there, because most of them cost more than an hour and none of them produce an obvious error message.

---

## 1. Hardware and addressing

| | Mini PC | Jetson |
|---|---|---|
| Hostname | `spring` | `manas-desktop` |
| User | `spring` | `manas` |
| Arch | x86_64 | aarch64 |
| OS | Ubuntu 22.04.5 | Ubuntu 22.04.5 |
| ROS | Humble | Humble |
| Role | LiDAR + robot_state_publisher | ZED 2i (tomorrow) |

### Interfaces on spring

| Interface | Address | Purpose |
|---|---|---|
| `wlp4s0` | `10.119.16.181/24` | Hotspot — SSH in from laptop, internet |
| `enp3s0` | `10.0.0.2/24` | Direct LAN cable to Jetson |
| `eno1` | `169.254.12.36/16` | Ouster LiDAR (DHCP link-local) |
| `docker0` | `172.17.0.1/16` | Unused, DOWN |

### Interfaces on manas

| Interface | Address | Purpose |
|---|---|---|
| `enP8p1s0` | `10.0.0.1/24` | Direct LAN cable to mini PC |

### Sensor

| | |
|---|---|
| Model | OS-1-32-U3 (32 beam, 90° vertical FOV) |
| Serial | 122220002210 |
| Firmware | v2.4.0 (2022) |
| Part number | 840-103575-06 |
| IP | `169.254.177.46` |

**`eno1` is DHCP link-local.** The `169.254.12.36` address is not static and can change. Always verify before launching, and see §5 if the LiDAR won't connect.

---

## 2. Architecture decision: no Zenoh

**We removed Zenoh. The two machines talk over plain DDS across the LAN cable.**

Rationale:

- Direct gigabit cable, one L2 segment, 0.4 ms ping. DDS handles this natively and well.
- Both machines were already on `ROS_DOMAIN_ID=0` and discovering each other directly over multicast. Zenoh was contributing nothing but noise.
- We only ever had **one** bridge running (on manas), dialing a stale address (`10.40.110.86`, from the campus-wifi era). Spring never had a bridge at all. The dual-PC comms that appeared to "work through Zenoh" were plain DDS the whole time.
- `zenoh_dds_setup.md` §21 warns against exactly this configuration — direct DDS between the two hosts you are bridging causes duplicate and looping traffic.

**When to revisit:** when a base station joins over WiFi. DDS degrades badly on lossy links; that is where Zenoh earns its keep. At that point set it up properly — bridge on *both* machines, different `ROS_DOMAIN_ID` per machine so DDS cannot cross the cable, and the router's connect endpoint pointing at a live address.

---

## 3. Time synchronisation (one-time setup, already done)

The Jetson has no RTC battery — its clock resets on every power cycle and it has no independent internet path. Spring becomes the time server.

Without this, every cross-machine `lookupTransform` throws extrapolation errors and TF appears broken for reasons that have nothing to do with TF.

**On spring:**

```bash
sudo apt install -y chrony
sudo tee -a /etc/chrony/chrony.conf <<'EOF'
allow 10.0.0.0/24
local stratum 10
EOF
sudo systemctl restart chrony
```

`local stratum 10` is not optional. Without it, spring refuses to serve time whenever its own upstream is unreachable — which is every time you test away from WiFi.

**On manas:**

```bash
sudo apt install -y chrony
sudo tee -a /etc/chrony/chrony.conf <<'EOF'
server 10.0.0.2 iburst prefer minpoll 2 maxpoll 4
EOF
sudo systemctl restart chrony
```

Installing chrony removes `systemd-timesyncd`. That is expected — they conflict.

**Verify:**

```bash
chronyc sources -v      # want '^*' next to 10.0.0.2
chronyc tracking        # want 'System time' under a few ms
```

Measured on 2026-09-13: **377 µs**, root delay 0.83 ms.

> Do **not** measure skew with `date; ssh other 'date'`. If SSH prompts for a password, you are measuring your typing speed. Use `chronyc tracking`.

---

## 4. Cold start sequence

### 4.1 SSH in

From your laptop, over the hotspot:

```bash
ssh spring@10.119.16.181
```

Start tmux immediately. The hotspot will drop and take every SSH session with it; tmux means you reconnect instead of restarting.

```bash
sudo apt install -y tmux    # first time only
tmux new -s robot
```

`Ctrl+B C` new window · `Ctrl+B 0/1/2` switch · `Ctrl+B D` detach · `tmux attach -t robot` to return.

Reach the Jetson through spring, over the cable:

```bash
ssh manas@10.0.0.1
```

### 4.2 Pre-flight checks (90 seconds, saves hours)

**On spring host:**

```bash
ip -br addr | grep -E 'eno1|enp3s0|wlp4s0'
ping -c 2 10.0.0.1              # Jetson
ping -c 2 169.254.177.46        # Ouster
```

Expected:

```
eno1      UP    169.254.12.36/16
enp3s0    UP    10.0.0.2/24
wlp4s0    UP    10.119.16.181/24
```

If `eno1` is `DOWN` → no carrier. Check the cable is in the right port and the Ouster interface box is powered. **This is a physical problem, nothing in software will fix it.**

If `eno1` is UP but has no address:

```bash
sudo dhclient -v eno1
ip -br addr show eno1
```

Note the address — you need it if you pin `udp_dest`.

Confirm the sensor is actually alive before blaming the driver:

```bash
curl -s http://169.254.177.46/api/v1/sensor/metadata/sensor_info
curl -s http://169.254.177.46/api/v1/sensor/config
```

The first should return JSON with `"status": "RUNNING"`. The second shows the sensor's *current* config, including the `lidar_mode` it is actually running.

**On manas:**

```bash
chronyc tracking | grep -E 'Reference ID|System time'
```

Want `10.0.0.2` and sub-millisecond.

### 4.3 Start the container

```bash
docker run -it --rm --network host -u root \
  -v ~/ouster_ws:/ouster_ws -w /ouster_ws \
  ouster_robot
```

`~/ouster_ws` is **bind-mounted from the host**, so the workspace survives `--rm`. Files can be edited from either side — but see §8 on ownership.

For a second shell into the same container:

```bash
docker ps --format '{{.Names}}'
docker exec -it <name> bash
```

Add `--name robot_core` to the `docker run` to stop having to look the name up.

### 4.4 Launch

```bash
cd /ouster_ws
source /opt/ros/humble/setup.bash
source install/setup.bash
ros2 launch agv_bringup state_publisher_launch.py
```

### 4.5 Activate the driver

The driver's built-in `auto_start` **does not work** (see §7). Transition it manually from a second shell:

```bash
cd /ouster_ws && source install/setup.bash
ros2 lifecycle set /ouster_driver configure
ros2 lifecycle set /ouster_driver activate
```

Both should report `Transitioning successful`.

### 4.6 Verify

```bash
ros2 node list
ros2 lifecycle get /ouster_driver
ros2 topic hz /points
ros2 run tf2_ros tf2_echo base_footprint os_lidar
```

**Expected:**

```
/launch_ros_<n>
/ouster_driver
/robot_state_publisher

active [3]

average rate: ~10.0

Translation: [0.100, 0.000, 0.356]
Rotation RPY (degree): [0.000, -0.000, 180.000]
```

Those numbers are the proof the whole chain is correct — see §6.

**From manas**, to confirm cross-machine DDS:

```bash
source /opt/ros/humble/setup.bash
ros2 daemon stop
ros2 topic list | grep -E 'tf|points'
ros2 run tf2_ros tf2_echo base_footprint os_lidar
```

---

## 5. The two files

### 5.1 `src/agv_bringup/urdf/robot.urdf.xacro`

```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro" name="robot">
  <xacro:arg name="lidar_x_offset" default="0.10"/>
  <xacro:arg name="lidar_y_offset" default="0.00"/>
  <xacro:arg name="lidar_z_offset" default="0.22"/>
  <xacro:arg name="cam_x_offset" default="0.305"/>
  <xacro:arg name="cam_y_offset" default="0.00"/>
  <xacro:arg name="cam_z_offset" default="0.08"/>

  <link name="base_footprint"/>
  <joint name="base_footprint_joint" type="fixed">
    <parent link="base_footprint"/>
    <child link="base_link"/>
    <origin rpy="0 0 0" xyz="0 0 0.1"/>
  </joint>

  <link name="base_link">
    <visual>
      <origin rpy="0 0 0" xyz="0 0 0.1"/>
      <geometry><box size="0.6 0.4 0.2"/></geometry>
      <material name="blue"><color rgba="0.2 0.2 1 1"/></material>
    </visual>
    <collision>
      <origin rpy="0 0 0" xyz="0 0 0.1"/>
      <geometry><box size="0.6 0.4 0.2"/></geometry>
    </collision>
  </link>

  <!-- Ouster mounting point. The driver hangs os_lidar and os_imu
       under os_sensor using the unit's own calibration.
       DO NOT define os_lidar or os_imu here. -->
  <joint name="lidar_joint" type="fixed">
    <parent link="base_link"/>
    <child link="os_sensor"/>
    <origin xyz="$(arg lidar_x_offset) $(arg lidar_y_offset) $(arg lidar_z_offset)" rpy="0 0 0"/>
  </joint>

  <link name="os_sensor">
    <visual>
      <geometry><cylinder length="0.04" radius="0.05"/></geometry>
      <material name="red"><color rgba="1 0 0 1"/></material>
    </visual>
    <collision>
      <geometry><cylinder length="0.04" radius="0.05"/></geometry>
    </collision>
  </link>

  <joint name="camera_joint" type="fixed">
    <parent link="base_link"/>
    <child link="camera_link"/>
    <origin xyz="$(arg cam_x_offset) $(arg cam_y_offset) $(arg cam_z_offset)" rpy="0 0 0"/>
  </joint>

  <link name="camera_link">
    <visual>
      <geometry><box size="0.010 0.03 0.03"/></geometry>
      <material name="red"><color rgba="1 0 0 1"/></material>
    </visual>
  </link>
</robot>
```

**The critical line is `<child link="os_sensor"/>`.** It used to be `laser_frame`. See §7.2.

### 5.2 `src/agv_bringup/launch/state_publisher_launch.py`

```python
import os
from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch.actions import OpaqueFunction, DeclareLaunchArgument
from launch.substitutions import LaunchConfiguration
from launch_ros.actions import LifecycleNode, Node
import xacro


def generate_launch_description():
    pkg_share = get_package_share_directory('agv_bringup')
    xacro_file = os.path.join(pkg_share, 'urdf', 'robot.urdf.xacro')

    return LaunchDescription([
        DeclareLaunchArgument('lidar_x', default_value='0.10'),
        DeclareLaunchArgument('lidar_y', default_value='0.00'),
        DeclareLaunchArgument('lidar_z', default_value='0.22'),
        DeclareLaunchArgument('cam_x', default_value='0.305'),
        DeclareLaunchArgument('cam_y', default_value='0.00'),
        DeclareLaunchArgument('cam_z', default_value='0.08'),
        DeclareLaunchArgument('sensor_hostname', default_value='169.254.177.46'),
        # spring's eno1 address. Empty also works (driver auto-detects).
        DeclareLaunchArgument('udp_dest', default_value='169.254.12.36'),
        # Empty = keep whatever mode the sensor is already configured for.
        DeclareLaunchArgument('lidar_mode', default_value=''),

        OpaqueFunction(function=lambda context: [

            Node(
                package='robot_state_publisher',
                executable='robot_state_publisher',
                name='robot_state_publisher',
                output='screen',
                parameters=[{
                    'robot_description': xacro.process_file(
                        xacro_file,
                        mappings={
                            'lidar_x_offset': context.perform_substitution(LaunchConfiguration('lidar_x')),
                            'lidar_y_offset': context.perform_substitution(LaunchConfiguration('lidar_y')),
                            'lidar_z_offset': context.perform_substitution(LaunchConfiguration('lidar_z')),
                            'cam_x_offset': context.perform_substitution(LaunchConfiguration('cam_x')),
                            'cam_y_offset': context.perform_substitution(LaunchConfiguration('cam_y')),
                            'cam_z_offset': context.perform_substitution(LaunchConfiguration('cam_z')),
                        }
                    ).toxml()
                }]
            ),

            LifecycleNode(
                package='ouster_ros',
                executable='os_driver',
                name='ouster_driver',
                namespace='/',
                output='screen',
                emulate_tty=True,
                parameters=[{
                    'sensor_hostname': context.perform_substitution(LaunchConfiguration('sensor_hostname')),
                    'udp_dest': context.perform_substitution(LaunchConfiguration('udp_dest')),
                    'lidar_mode': context.perform_substitution(LaunchConfiguration('lidar_mode')),
                    'proc_mask': 'PCL|IMU',
                    'sensor_frame': 'os_sensor',
                    'lidar_frame': 'os_lidar',
                    'imu_frame': 'os_imu',
                    'point_cloud_frame': 'os_lidar',
                    'pub_static_tf': True,
                    # NOTE: does not work on this driver build. Transition
                    # manually — see §4.5 and §7.1.
                    'auto_start': True,
                }]
            ),
        ])
    ])
```

With `namespace='/'`, topics are `/points`, `/imu`, `/scan` — **not** `/ouster/points`. The `/ouster/*` namespace belongs to upstream's `sensor.launch.xml`, which we do not use.

### 5.3 Recommended upgrade (untested)

`auto_start` is broken, so the launch file should drive the lifecycle itself. This is the pattern Nav2 uses and it does not depend on the driver's internal logic at all. **Not yet verified on our stack.**

```python
from launch.actions import EmitEvent, RegisterEventHandler
from launch.events import matches_action
from launch_ros.events.lifecycle import ChangeState
from launch_ros.event_handlers import OnStateTransition
import lifecycle_msgs.msg

driver = LifecycleNode(...)   # drop 'auto_start' from parameters

configure = EmitEvent(event=ChangeState(
    lifecycle_node_matcher=matches_action(driver),
    transition_id=lifecycle_msgs.msg.Transition.TRANSITION_CONFIGURE))

activate_on_inactive = RegisterEventHandler(OnStateTransition(
    target_lifecycle_node=driver,
    goal_state='inactive',
    entities=[EmitEvent(event=ChangeState(
        lifecycle_node_matcher=matches_action(driver),
        transition_id=lifecycle_msgs.msg.Transition.TRANSITION_ACTIVATE))]))

# return ordered: [rsp, driver, activate_on_inactive, configure]
```

---

## 6. Reading the TF output

```
base_footprint → os_lidar
Translation: [0.100, 0.000, 0.356]
Rotation RPY (degree): [0.000, -0.000, 180.000]
```

**Z = 0.356** decomposes as:

| Source | Value | Where it comes from |
|---|---|---|
| `base_footprint → base_link` | 0.100 | URDF, hand-measured |
| `base_link → os_sensor` | 0.220 | URDF, hand-measured |
| `os_sensor → os_lidar` | **0.036** | **the sensor's own calibration** |

**Yaw = 180°** also comes from the sensor. The Ouster's optical frame is rotated half a turn relative to its housing.

This is why `pub_static_tf: true` matters and why you must never hand-write `os_sensor → os_lidar`. The 36 mm and the 180° are read from `lidar_to_sensor_transform` in the unit's metadata — they are per-unit calibration. Hardcoding them would mirror every pointcloud front-to-back, putting the left lane on the right. That bug survives all the way to a competition run.

**Verify your hand-measured numbers with a tape measure.** For AutoNav, ground-plane removal and lane extraction are directly sensitive to LiDAR height; a 3 cm error costs more debugging time than everything in this document.

---

## 7. Traps — every one of these cost us an hour

### 7.1 `autostart` is not a parameter. It is `auto_start`.

**Symptom:** driver sits in `unconfigured` forever, publishes nothing, logs no error about the parameter.

**Cause:** ROS 2 silently discards parameter overrides for undeclared names. No warning. The declared name is `auto_start`:

```
ouster-ros/src/os_sensor_node.cpp:100:    declare_parameter("auto_start", false);
```

Note the default differs per node — `os_replay_node` and `os_pcap_node` declare it `true`, but `os_sensor_node` (which `os_driver` composes) declares it `false`.

**Status:** even with the correct spelling, auto-start fails on our build:

```
[ERROR] [ouster_driver]: Service /os_driver/change_stateis not available.
```

The driver builds a *relative* service client `os_driver/change_state`, which resolves against its own namespace. Upstream's `sensor.launch.xml` uses namespace `/ouster` and node name `os_driver`, so it resolves correctly there. Renaming our node to `os_driver` did **not** fix it — the logger name stayed `ouster_driver`, suggesting the name is baked in somewhere in the C++ rather than derived from the node.

**Workaround:** transition manually (§4.5) or move to explicit lifecycle events (§5.3).

**General rule: a parameter you set that has no effect is indistinguishable from a parameter that does not exist.** Always confirm with `ros2 param get <node> <name>`.

### 7.2 `lidar_frame` will hijack your URDF frame

**Symptom:** LiDAR silently detaches from the robot the moment the driver activates. Pointclouds become untransformable to `base_link`.

**Cause:** we had `'lidar_frame': 'laser_frame'` in the launch file. `ouster_ros` broadcasts `os_sensor → <lidar_frame>`. With that setting it publishes `os_sensor → laser_frame`, while the URDF publishes `base_link → laser_frame`.

**A frame can have only one parent.** tf2's static buffer is keyed by child frame and last-writer-wins. Whichever message arrives second silently overwrites the first.

The four real frame parameters, from `os_static_transforms_broadcaster.h:42-46`:

```cpp
declare_parameter("sensor_frame", "os_sensor");
declare_parameter("lidar_frame", "os_lidar");
declare_parameter("imu_frame", "os_imu");
declare_parameter("point_cloud_frame", "");   // NB: not os_lidar, despite the YAML comment
declare_parameter("pub_static_tf", true);
```

`laser_frame` is **not** a parameter. Setting it does nothing.

**Rule:** leave the driver's frame names at their defaults and adopt `os_sensor` into the URDF as the mounting point.

### 7.3 Do not publish the same transform twice

We had four publishers on `/tf_static`: `robot_state_publisher`, `ouster_driver`, plus `lidar_static_bridge` and `camera_static_bridge` — two `static_transform_publisher` nodes duplicating transforms the URDF already provided.

They were added to work around `ros2 topic echo /tf_static` printing nothing. **That was not an empty topic** — see §9.1.

Both sources happened to publish identical values, so it was silently harmless. That is what makes it dangerous: the day someone edits the URDF offsets without editing the launch defaults, you get a transform that flips depending on node startup order.

**Rule: exactly one publisher per parent→child pair.** Check with:

```bash
ros2 topic info /tf_static --verbose | grep -c "Node name"
```

Should be **2** (robot_state_publisher + driver).

### 7.4 `--symlink-install` does not retrofit an existing build

**Symptom:** you edit a launch file, relaunch, and nothing changes. `grep` confirms your edit is in `src/`.

**Cause:** the flag only takes effect when a package is configured from scratch. If `build/` and `install/` were created by an earlier plain `colcon build`, re-running with the flag still copies.

**Diagnose:**

```bash
ls -la install/agv_bringup/share/agv_bringup/launch/
```

A symlink shows `-> /ouster_ws/src/...`. A real file means you are launching a stale copy.

**Fix:**

```bash
rm -rf build/agv_bringup install/agv_bringup
colcon build --packages-select agv_bringup --symlink-install
```

**Related:** the `robot_description` parameter dump shows which file xacro actually read. If it says `install/...` and you have been editing `src/...`, this is your problem.

### 7.5 Two drivers, one sensor

At one point we had both our launch and upstream's `sensor.launch.xml` running. The working pointcloud was coming from upstream's `/ouster/os_driver` while our `/ouster_driver` sat in `unconfigured`. The TF chain that appeared to work was two launches accidentally cooperating.

An Ouster accepts UDP destination config from whichever client asked most recently. Two drivers will overwrite each other and produce a stream that works, stops, then works again — the most miserable bug class to chase at a competition.

**Check before trusting any result:**

```bash
ros2 node list
ros2 lifecycle nodes
ps aux | grep -E 'os_driver|ros2 launch' | grep -v grep
```

The `TTY` column in `ps` groups processes by which terminal launched them — that is how you tell whose launch is whose.

### 7.6 Container PIDs are not host PIDs

`kill <pid>` from the host will not find a PID you read from `ps aux` inside the container. Separate PID namespaces. `/dev/pts/N` is likewise unrelated between host and container.

Kill from inside the container, or re-find the host-side PID with `ps aux | grep ...` on the host.

### 7.7 Things that live only in an ephemeral container

We run with `--rm`. `ros-humble-xacro` was installed by hand and vanished on exit, producing:

```
ModuleNotFoundError: No module named 'xacro'
```

The **workspace** is safe — it is bind-mounted from `~/ouster_ws` on the host. Anything `apt install`ed is not.

**Fix properly:** add it to the Dockerfile in `~/ouster_docker`. **Fix quickly:**

```bash
docker commit <container> ouster_robot:working-2026-09-13
```

### 7.8 Mixed file ownership

Three users have written into `~/ouster_ws`: `spring` (host), `root` (container), `robot` (image default). Result is silent write failures — "error writing" in nano, or `colcon build` failing unpredictably.

```bash
sudo chown -R spring:spring ~/ouster_ws
```

Pick one: always edit from the host as `spring`, or always from the container as root. Mixing produces files you think you changed but did not.

### 7.9 Link-local addressing

`169.254.x.x` is APIPA — an address a machine assigns itself when DHCP fails. It works only if **both** ends have a link-local address on the same segment.

Spring had no `169.254.x` address at all at one point because `eno1` was `DOWN` (no carrier), so the Ouster was unreachable by any path. No amount of driver configuration fixes a cable.

**Consider moving to a static subnet on `eno1`** and configuring the sensor to match. Link-local addresses can change across reboots, which means `udp_dest` can go stale.

---

## 8. Debugging playbook

### 8.1 "robot_state_publisher isn't publishing transforms"

Almost certainly it is. Check in this order:

**All your joints are `type="fixed"`.** RSP splits its output: fixed joints → `/tf_static` (published once, latched); movable joints → `/tf` (continuous, driven by `/joint_states`). With no movable joints, **`/tf` will be empty forever.** That is arithmetic, not a bug. `/joint_states` having no publisher is also fine.

**`ros2 topic echo /tf_static` lies to you.** `/tf_static` is `TRANSIENT_LOCAL` (latched). A plain echo subscribes as `VOLATILE` and shows *nothing* even when everything is correct:

```bash
ros2 topic echo /tf_static --qos-durability transient_local --qos-reliability reliable --once
```

**`tf2_monitor` output looks insane for static-only trees.** All of this is normal:

```
published by <no authority available>     ← tf2 records no broadcaster for statics
119837 Hz                                 ← divide by the ~0 gap between simultaneous statics
Average Delay: 1334.6                     ← now − stamp; statics are stamped once at launch
```

**`tf2_echo` prints a warning then works.** This is a startup race — it calls `canTransform` before the latched data arrives, logs once, then succeeds. The warning appearing *above* successful output is expected.

### 8.2 Standard diagnostic sweep

```bash
# Graph
ros2 node list
ros2 topic list
ros2 topic info /tf_static --verbose        # count publishers — want 2
ros2 topic info /tf --verbose

# Lifecycle
ros2 lifecycle nodes
ros2 lifecycle get /ouster_driver

# Parameters — confirm what you set actually took
ros2 param get /ouster_driver sensor_hostname
ros2 param get /ouster_driver udp_dest
ros2 param get /ouster_driver lidar_frame
ros2 param list /ouster_driver | grep -i frame

# TF
ros2 run tf2_ros tf2_echo base_footprint os_lidar
ros2 run tf2_ros tf2_monitor
ros2 run tf2_tools view_frames

# Data
ros2 topic hz /points
ros2 topic echo /points --field height --once    # 32 for OS-1-32

# Processes
ps aux | grep -E 'os_driver|ros2 launch' | grep -v grep
```

Note `ros2 daemon stop` before `node list` after lots of restarting — the daemon caches dead nodes and shows ghosts.

### 8.3 Decision tree

| Symptom | Check |
|---|---|
| `tf2_echo` says frame does not exist | Is the driver `active`? Only it publishes `os_lidar`. |
| Driver stuck `unconfigured` | `auto_start` (§7.1). Transition manually. |
| `configure` fails | `ping` and `curl` the sensor. Check `lidar_mode` against `/api/v1/sensor/config`. |
| `active` but no points | `udp_dest` on a multi-homed host. Pin it, and pin `lidar_port: 7502`, `imu_port: 7503`. |
| Edits have no effect | §7.4 — stale `install/`. |
| Transform has wrong values | Count publishers on `/tf_static` (§7.3). |
| Works, then stops, then works | Two drivers (§7.5). |
| Cross-machine TF extrapolation errors | `chronyc tracking` on both. |
| `ModuleNotFoundError` | §7.7 — ephemeral container. |
| Cannot save a file | §7.8 — ownership. |

### 8.4 Sequencing rule

Do not debug two things at once. Get the LiDAR fully working on one machine with plain DDS before touching cross-machine transport. Most of 2026-09-13 was spent on a Zenoh configuration that turned out not to be load-bearing at all.

---

## 9. Known issues

### 9.1 Points arriving at ~7 Hz instead of 10

Measured after switching to our own launch:

```
average rate: 7.1    min: 0.099s    max: 0.301s    std dev: 0.070
```

The sensor rotates at 10 Hz and an earlier run measured a clean 9.99 Hz with 0.6 ms jitter. A max interval of exactly 3× the 100 ms period means we are **dropping roughly every third scan**, not running slow.

Candidates, in order:
1. UDP receive buffer too small — try raising `net.core.rmem_max`
2. Random port assignment (`lidar port set to zero` warning) — pin `lidar_port: 7502`, `imu_port: 7503`
3. Interface contention now that all three NICs are active

Not yet investigated. A working baseline is worth more than an optimised broken one.

### 9.2 `auto_start` unresolved

See §7.1. Workaround in place. Move to explicit lifecycle events (§5.3).

### 9.3 `scan_ring` unsuitable for a 32-beam U3

`SCAN` was dropped from `proc_mask`. The default `scan_ring: 0` takes beam 0, which on a 90°-FOV U3 points steeply off-axis — at the ground or the sky. If a LaserScan is ever needed, pick a near-horizontal ring (around 15–16 on a 32-beam unit) and verify in RViz.

For AutoNav you want the full pointcloud and proper ground-plane removal, not a single-ring LaserScan.

### 9.4 Old firmware

FW v2.4.0 (2022). Several parameters in `driver_params.yaml` are gated on FW 3.1+, 3.2+ or 4.0+ (`gyro_fsr`, `columns_per_packet`, `min_distance`, `return_order`). We set none of them. Do not add them, and do not be surprised when documented features are absent.

---

## 10. Corrections to existing repo docs

| Doc | Says | Reality |
|---|---|---|
| `LAN_cable_setup.md` | `10.50.0.1` / `10.50.0.2` | now `10.0.0.1` / `10.0.0.2` |
| `zenoh_dds_setup.md` | `-e tcp/10.40.110.86:7447` | stale campus-wifi address; manas's bridge was dialing it with nothing there |
| both | Zenoh is the transport | Zenoh removed; plain DDS over the cable (§2) |

**This drift is what made 2026-09-13 confusing.** The docs described a topology that no longer existed, so every diagnosis started from a false premise. Update the IPs in those two files, or add a pointer to this document.

---

## 11. Quick reference

```bash
# ── laptop ──
ssh spring@10.119.16.181
tmux new -s robot            # or: tmux attach -t robot

# ── spring host ──
ip -br addr | grep -E 'eno1|enp3s0|wlp4s0'
ping -c 2 169.254.177.46
curl -s http://169.254.177.46/api/v1/sensor/metadata/sensor_info

docker run -it --rm --network host -u root \
  -v ~/ouster_ws:/ouster_ws -w /ouster_ws ouster_robot

# ── container, shell 1 ──
cd /ouster_ws
source /opt/ros/humble/setup.bash
source install/setup.bash
ros2 launch agv_bringup state_publisher_launch.py

# ── container, shell 2 ──
docker exec -it <name> bash
cd /ouster_ws && source install/setup.bash
ros2 lifecycle set /ouster_driver configure
ros2 lifecycle set /ouster_driver activate
ros2 topic hz /points
ros2 run tf2_ros tf2_echo base_footprint os_lidar

# ── after any launch/urdf edit ──
colcon build --packages-select agv_bringup --symlink-install
source install/setup.bash

# ── save container state ──
docker commit <name> ouster_robot:working-YYYY-MM-DD
sudo chown -R spring:spring ~/ouster_ws    # on host
```

---

