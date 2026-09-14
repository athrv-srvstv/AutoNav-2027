# ZED 2i TF Integration + LiDAR Tuning — 2026-09-14

**Status:** working. Full TF tree resolves identically on both machines.
**Companion doc:** `lidar_bringup.md` (2026-09-13) — read that first for the LiDAR itself.
**Supersedes:** node name `/ouster_driver` in that doc is now `/os_driver` (§1.1).

Day 2. The goal was the ZED 2i transform tree. Roughly half the session went to problems that had nothing to do with the camera — a stale shell config, and a measurement technique that was itself causing the fault it appeared to measure. Both are recorded here because both will recur.

---

## 1. What changed since 2026-09-13

### 1.1 Node rename landed — `/ouster_driver` → `/os_driver`

The rename was made late on day 1 but never took effect, because `install/` held a stale copy (day-1 doc §7.4). After a clean rebuild it applied.

**Consequence: `auto_start` now works.** The first `ros2 lifecycle set /os_driver configure` returned:

```
Unknown transition requested, available ones are:
- deactivate [4]
- shutdown [7]
```

The node was already `active` — it self-started. Day-1 §7.1 is closed: the driver's auto-start builds a *relative* service client `os_driver/change_state`, so the node must be **named** `os_driver` for it to find its own lifecycle service. The manual `configure`/`activate` steps are no longer needed.

The log prefix still reads `[os_driver]` — the logger name is baked into the C++ and does not follow the ROS node name. Do not use the log prefix to identify a node.

### 1.2 LiDAR mode reduced

`lidar_mode` was `''` (inherit), and the sensor was sitting at `1024x10` with the `LEGACY` UDP profile. Now pinned to `512x10`. See §3 for why this turned out not to be the fix for anything.

### 1.3 URDF: `camera_link` replaced by `zed2i_camera_link`

The placeholder `camera_link` was deleted and replaced with a real mounting joint for the ZED. See §4.

---

## 2. Trap: a stale `.bashrc` broke ROS entirely

**Symptom**, on spring, inside the container, before any node started:

```
ros2: 192.168.123.22: does not match an available interface
[ERROR] [rmw_cyclonedds_cpp]: rmw_create_node: failed to create domain, error Error
error creating node: rcl node's rmw handle is invalid
```

**Cause.** Spring's host `~/.bashrc` had gained two lines between sessions:

```bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
export CYCLONEDDS_URI=~/cyclonedds.xml
```

Docker inherits the host environment, so the container got both. `~/cyclonedds.xml` contained:

```xml
<NetworkInterface address='192.168.123.22' />
<AllowMulticast>true</AllowMulticast>
<DontRoute>true</DontRoute>
```

`192.168.123.x` is the **Unitree robot dog's** default subnet. The file was copied from an unrelated project. It pins CycloneDDS to an interface that exists on neither machine, and `DontRoute` compounds it.

**Worse than the crash:** manas had `RMW_IMPLEMENTATION` empty (→ FastDDS) while spring was on CycloneDDS. **Different RMW implementations cannot communicate at all.** Even with a valid interface address, the two machines would have been mutually invisible.

**Fix.** On spring's host, comment out both lines and move the file aside:

```bash
cp ~/.bashrc ~/.bashrc.bak
# comment out the RMW_IMPLEMENTATION and CYCLONEDDS_URI exports
mv ~/cyclonedds.xml ~/cyclonedds.xml.unitree-stale
```

Then **exit and restart the container.** `unset` in a shell does not fix a container whose environment was already poisoned — launch-spawned processes inherit the original env.

**Diagnostic sweep for this class of problem:**

```bash
echo "RMW=$RMW_IMPLEMENTATION"
echo "URI=$CYCLONEDDS_URI"
env | grep -i -E 'cyclone|rmw|fastrtps'
grep -n "source\|export" ~/.bashrc | grep -v "^#"
```

That last command found a second stale item in the same file — a `source` of `~/self_drive/install/...` for a workspace that no longer exists, producing an error on every new shell.

**Rule: read the whole of `~/.bashrc` once, deliberately, rather than discovering its contents one failure at a time.** Three pieces of stale config have now been found across two days: this DDS file, the dead workspace, and the Zenoh IPs in the repo docs.

**Both machines must use the same RMW.** Current state: both unset/default → FastDDS. `ROS_DOMAIN_ID=0` on both (spring sets it explicitly at `.bashrc:127`, manas defaults).

---

## 3. Trap: `ros2 topic hz` on a remote machine stalls the publisher

This consumed over an hour and the conclusion is that **the LiDAR was never broken.**

### The symptom

Spring's launch console filled with:

```
[WARN] [lidar_packet_hander]: lidar_scans 70% full, THROTTLING
[WARN] [lidar_packet_hander]: lidar_scans full, DROPPING PACKET
```

Spring reported a clean `10.002 Hz` on `/points`. Manas reported `~5.3 Hz` for the same topic.

### What was ruled out, and how

| Hypothesis | Test | Result |
|---|---|---|
| CPU saturation | `top -bn1 \| head -15` and `nproc` | 16 cores, 98.9% idle, load 0.09. **Not CPU.** |
| Network link fault | `sudo ethtool enp3s0 \| grep -E 'Speed\|Duplex'` on both | 1000Mb/s full duplex both ends. **Not the link.** |
| Packet loss on the wire | `ip -s link show enp3s0` | 0 errors, 0 dropped, 0 missed. **Not the wire.** |
| Socket buffers too small | raised `net.core.rmem_max` to 26214400 on both | No change. **Not the buffers.** |
| Data volume | reduced `1024x10` → `512x10` | **No change.** Halving the data did nothing. |

That last row is the decisive one. If volume were the constraint, halving it would have helped measurably. It did not.

### The actual cause

The Ouster driver publishes `/points` with **reliable** QoS. Reliable means every message must be acknowledged by every matched subscriber. When manas could not keep up, the publisher blocked waiting on retransmit ACKs; the publish call stalled; the driver's processing thread stalled behind it; and the driver's internal `lidar_scans` queue — the buffer between its packet-receive thread and its processing thread — backed up and dropped packets.

`ros2 topic hz` on spring still read 10 Hz because it measures messages *handed to* the publisher, not messages that completed transmission.

### The confirming test

Ctrl+C the `ros2 topic hz /points` on manas and watch spring's console. THROTTLING stopped immediately and did not resume.

### The lesson

**Measuring a system can change its behaviour.** `ros2 topic hz` on a remote machine is not a passive observation — it creates a reliable-QoS subscriber that can throttle the publisher. This is the observer effect in a form that is easy to mistake for a performance bug.

### The resolution

For this architecture, **manas must never subscribe to `/points`.** FAST-LIO2 runs on spring where the LiDAR is. There is no reason for ~10 MB/s of point cloud to cross the cable, ever. The remote subscription was a test, the test damaged the thing it measured, and the correct response is to stop doing it.

If clouds are ever genuinely needed on manas, switch the publisher to best-effort so a slow subscriber cannot block it:

```python
'use_system_default_qos': False,
```

Best-effort drops rather than blocks. For sensor data this is correct anyway — a stale point cloud is worthless, so retransmitting it is pure harm.

---

## 4. Trap: sensor settings persist in firmware after you remove them from config

The nastiest failure of the two days, because it breaks the assumption that your launch file describes your system.

### Sequence

1. Added `'udp_profile_lidar': 'RNG19_RFL8_SIG16_NIR16'` to the launch file
2. Driver crashed:
   ```
   terminate called after throwing an instance of 'std::out_of_range'
     what():  Field 'WINDOW' not found in LidarScan.
   ```
3. **Removed the line** from the launch file, rebuilt, relaunched
4. **Driver crashed identically.** Startup banner still read `lidar udp profile: RNG19_RFL8_SIG16_NIR16`

### Why

An absent `udp_profile_lidar` means *"leave the sensor as it is."* The sensor had persisted the profile in its own firmware. Removing the line did not revert it — it just stopped asking, while the sensor kept the broken setting.

`curl -s http://169.254.177.46/api/v1/sensor/config` confirmed the sensor was still on `RNG19_RFL8_SIG16_NIR16` with nothing in the launch file requesting it.

### The crash itself

`WINDOW` is a field belonging to the `RNG15_RFL8_WIN8` profile. The driver's field lookup queried it against a profile that does not provide it. Firmware is **v2.4.0 (2022)**; the bundled SDK is **0.16.2**. The two disagree about profile field tables.

### Fix

Per-key config POST is **not supported on FW 2.4.0**:

```bash
curl -X POST -H "Content-Type: application/json" -d '"LEGACY"' \
  http://169.254.177.46/api/v1/sensor/config/udp_profile_lidar
# → {"error": {"title": "404 Not Found"}}
```

Set it explicitly from the launch file instead:

```python
'udp_profile_lidar': 'LEGACY',
```

This is better practice regardless: the setting lives in version control where teammates can see it, rather than as invisible state inside a sensor.

### Rules that follow

- **Read the startup banner every launch.** Treat it, not your launch file, as ground truth.
- **Pin values explicitly rather than leaving them unspecified**, when a wrong value can be persisted.
- Consider `'persist_config': False` so sensor-side settings do not survive power cycles invisibly.
- Delete stale metadata when changing profiles — the driver caches it:
  ```bash
  find /ouster_ws -name "*metadata*.json" -delete
  ```

### Running tally of this driver's failure modes

Four parameters, four *different* ways of failing:

| Parameter | Failure mode |
|---|---|
| `autostart` | silently ignored (undeclared name; real name is `auto_start`) |
| `lidar_frame` | silently destructive (hijacks a URDF frame) |
| `udp_profile_lidar` | hard crash, **and persists in hardware after removal** |
| `point_type: xyzir` | suspected crash (see below) |

**This codebase does not validate its inputs.** Change one parameter at a time and check the startup banner after every launch.

> **Note on `point_type`:** `xyzir` was added and removed in the same session as `udp_profile_lidar`. Because the profile persisted in firmware, the crash continued after `point_type` was reverted — so `xyzir` was **never actually cleared of blame**. It may well be fine with the `LEGACY` profile. If you want the bandwidth saving (~16 bytes/point vs 48 for `original`), retry it now that the profile is stable, **as the only change**.

---

## 5. The ZED 2i integration

### 5.1 Wrapper facts established

```bash
ls ~/zed-ros2-wrapper/zed_wrapper/config/
# common_stereo.yaml present → wrapper v4.2+
```

| Property | Value |
|---|---|
| Config files | `common_stereo.yaml` (shared) + `zed2i.yaml` (camera-specific) |
| Camera name | `zed2i` |
| Namespace | `/zed2i` |
| Its RSP node | `zed2i_state_publisher` |
| Its description topic | `/zed2i_description` (remapped, **not** `/robot_description`) |
| Container workspace | `/root/ros2_ws` — **separate from `~/zed-ros2-wrapper` on the host** |

The remapping at `zed_camera.launch.py:385` matters:

```python
remappings=[('robot_description', camera_name_val+'_description')]
```

It means the ZED's RSP will **not** collide with spring's RSP. Different node name, different topic. The two coexist cleanly — which is what makes the split-URDF architecture viable.

### 5.2 Frame naming — not what you would guess

The ZED's segments, from its RSP startup log:

```
zed2i_camera_center
zed2i_camera_link
zed2i_left_camera_frame
zed2i_left_camera_frame_optical      ← note the word order
zed2i_right_camera_frame
zed2i_right_camera_frame_optical
```

**It is `_frame_optical`, not `_optical_frame`.** The words are transposed relative to the common ROS convention and relative to older ZED wrapper versions. Querying the wrong one produces `Invalid frame ID ... frame does not exist`, which reads like a missing transform rather than a typo.

**Always get frame names from the RSP's own startup log**, not from memory or documentation.

### 5.3 This wrapper version has no mounting arguments

```bash
ros2 launch zed_wrapper zed_camera.launch.py --show-args
```

The full argument list contains `publish_urdf`, `publish_tf`, `publish_map_tf`, `publish_imu_tf` — but **no `cam_pos_x`, no `cam_roll`, no `base_frame`.**

Confirmed by reading the xacro invocation (`zed_camera.launch.py:346-370`): it passes only `camera_name`, `camera_model`, and optionally GNSS coordinates. Nothing about mounting.

**Consequence:** the ZED's URDF is camera-only, rooted at `zed2i_camera_link` with no parent. The mounting joint must be supplied by spring's URDF. This decided the architecture.

### 5.4 `publish_tf:=false` is mandatory, not a preference

From `--show-args`:

> `publish_tf`: Enable publication of the **`odom -> camera_link`** TF

Note the child: `camera_link`, not `base_link`.

If left enabled, the ZED publishes `odom → zed2i_camera_link` while spring publishes `base_link → zed2i_camera_link`. **That frame then has two parents.** tf2's static buffer is keyed by child frame and last-writer-wins, so the camera silently reparents to a floating `odom` and the tree splits.

This is exactly the `laser_frame` trap from day-1 §7.2, in a new costume.

```bash
publish_tf:=false      # required
publish_map_tf:=false  # ignored when publish_tf is false, but be explicit
```

Independently, **FAST-LIO2 will own `odom → base_link`.** Two writers on a *moving* transform produce a robot that visibly jitters between poses — far harder to diagnose than a static conflict.

### 5.5 Architecture: split URDF, one owner per edge

```
base_footprint
 └─ base_link
     ├─ os_sensor ──────────→ os_lidar, os_imu    ← Ouster driver (spring)
     └─ zed2i_camera_link ──→ zed2i_camera_center ← ZED wrapper RSP (manas)
                                ├─ left_camera_frame → _frame_optical
                                └─ right_camera_frame → _frame_optical
```

| Edge | Owner | Machine | Source |
|---|---|---|---|
| `base_footprint → base_link` | agv_bringup RSP | spring | hand-measured |
| `base_link → os_sensor` | agv_bringup RSP | spring | hand-measured |
| `os_sensor → os_lidar/os_imu` | ouster driver | spring | **factory calibration** |
| `base_link → zed2i_camera_link` | agv_bringup RSP | spring | hand-measured |
| everything below `zed2i_camera_link` | zed2i RSP | manas | **factory calibration** |
| `odom → base_link` | *(vacant)* | spring | FAST-LIO2, next |

**Principle, applied twice now: let each driver publish what it calibrates, and adopt its root frame into your own URDF as the mounting point.** Hand-writing `os_sensor → os_lidar` would have cost a 36 mm offset and a 180° yaw. Hand-writing the ZED's internals would have cost the stereo baseline and the optical-frame rotation.

### 5.6 URDF changes

Added to `~/ouster_ws/src/agv_bringup/urdf/robot.urdf.xacro`, replacing the placeholder `camera_link`/`camera_joint` blocks:

```xml
  <!-- ZED 2i mounting point. The wrapper's own RSP on manas publishes
       everything below zed2i_camera_link using factory calibration.
       Do NOT define any zed2i_* frames here except this one. -->
  <joint name="zed_joint" type="fixed">
    <parent link="base_link"/>
    <child link="zed2i_camera_link"/>
    <origin xyz="$(arg cam_x_offset) $(arg cam_y_offset) $(arg cam_z_offset)" rpy="0 0 0"/>
  </joint>

  <link name="zed2i_camera_link"/>
```

The link is intentionally empty — no visual, no collision. The ZED's own URDF supplies the geometry. Defining a visual here would render two overlapping camera bodies in RViz.

The `cam_x_offset` / `cam_y_offset` / `cam_z_offset` xacro args are retained at the top of the file.

### 5.7 Working launch command

```bash
ros2 launch zed_wrapper zed_camera.launch.py \
  camera_model:=zed2i \
  camera_name:=zed2i \
  publish_tf:=false \
  publish_map_tf:=false \
  publish_urdf:=true
```

**Put this in a shell script on manas.** `publish_tf:=false` is load-bearing and relying on memory to type it every time is how the two-parents bug gets reintroduced.

---

## 6. Verification — and how to read it

### Commands

```bash
# On manas
ros2 run tf2_ros tf2_echo base_footprint zed2i_left_camera_frame_optical
ros2 run tf2_ros tf2_echo os_lidar zed2i_left_camera_frame_optical

# On spring — must produce identical numbers
ros2 run tf2_ros tf2_echo base_footprint zed2i_left_camera_frame_optical
```

### Results, 2026-09-14

**`base_footprint → zed2i_left_camera_frame_optical`** — identical on both machines:

```
Translation: [0.295, 0.060, 0.195]
Rotation RPY (degree): [-90.000, -0.000, -90.000]
```

Decomposed:

| Axis | Value | Arithmetic |
|---|---|---|
| X | 0.295 | 0.305 (mount) − 0.010 (left sensor is 10 mm behind camera center) |
| Y | 0.060 | half the 120 mm stereo baseline |
| Z | 0.195 | 0.100 (footprint→base_link) + 0.080 (mount) + 0.015 (camera_link→center) |

The **−90° / 0 / −90°** rotation is the optical frame convention: Z forward, X right, Y down. Every vision library expects this. It is present automatically because the wrapper owns its own frames.

**`os_lidar → zed2i_left_camera_frame_optical`** — the LiDAR-to-camera extrinsic:

```
Translation: [-0.195, -0.060, -0.161]
Rotation RPY (degree): [-90.000, -0.000, 90.000]
```

Cross-check: from `base_footprint`, the LiDAR is at z=0.356 and the camera at z=0.195. Difference = 0.161. Matches the Z component exactly.

**This is the number that matters for sensor fusion** — projecting lane detections into the point cloud, or colourising points with image data. Composed from two URDFs, two factory calibrations, and two machines.

### Why "identical on both machines" is the real test

A transform resolving on the machine that publishes it proves very little. Resolving identically on the *other* machine proves the whole chain: DDS transport, matching RMW, `/tf_static` transient-local delivery, clock sync, and that no frame has two competing parents.

---

## 7. Debugging techniques that worked

Collected from both days. The pattern worth internalising: **each of these turns an ambiguous symptom into a specific fact.**

### Reading `tf2_echo` errors precisely

Three distinct messages, three different problems:

| Message | Means |
|---|---|
| `Invalid frame ID "X" ... target_frame` | Frame X is not in the buffer at all |
| `Invalid frame ID "Y" ... source_frame` | X exists now, Y does not — **progress, the error moved** |
| `Could not find a connection ... not part of the same tree` | **Both frames exist.** The joining edge is missing |

Watching *which* frame the error names, and whether it changes between attempts, is the fastest way to localise a broken tree. The third message specifically told us spring's tree and the ZED's tree were both present and only the bridging joint was absent.

A warning that appears **once, above** successful output is a startup race, not a failure — `tf2_echo` calls `canTransform` before latched data arrives.

### Reading the RSP startup log as ground truth

```
[robot_state_publisher]: got segment base_footprint
[robot_state_publisher]: got segment base_link
[robot_state_publisher]: got segment os_sensor
[robot_state_publisher]: got segment zed2i_camera_link
```

This is the authoritative list of what a URDF actually produced. It answered two questions this session: whether the URDF edit reached the running process, and what the ZED's real frame names are (§5.2).

### Verifying that an edit actually took effect

Three separate places a change can fail to land:

```bash
grep -n "name='os_driver'" src/agv_bringup/launch/state_publisher_launch.py   # did it save?
ls -la install/agv_bringup/share/agv_bringup/launch/                          # symlink or stale copy?
# then: the RSP log or the driver banner                                      # did the process see it?
```

> **`grep -c` counts lines, not occurrences.** Comments mentioning a term inflate the count. Use `grep -n` and read the output rather than matching an expected number — twice across these two days an expected count was wrong for exactly this reason.

### Distinguishing nodes from topics

`/os_driver` is a **node**. `/points` is a **topic**. Separate registries; `ros2 topic info /os_driver` correctly reports "Unknown topic". They share the slash-prefixed style, which is why this confuses.

```bash
ros2 node info /os_driver     # everything this node publishes/subscribes
ros2 topic list               # what topics exist
```

`ros2 node info` is the reliable way to find a node's real topic names without guessing.

### Identifying which process owns what

```bash
ps aux | grep -E 'os_driver|ros2 launch' | grep -v grep
```

The **TTY column** (`pts/0`, `pts/3`) groups processes by the terminal that launched them. That is how day 1 established that a working point cloud was coming from a stray upstream launch rather than our own.

With tmux:

```bash
tmux list-panes -a -F '#{window_index}.#{pane_index} #{pane_current_command} #{pane_pid}'
```

**Container PIDs are not host PIDs.** Separate namespaces — `kill <pid>` from the host will not find a PID read inside a container. `/dev/pts/N` is likewise unrelated between the two.

### Querying the sensor directly, bypassing ROS entirely

```bash
curl -s http://169.254.177.46/api/v1/sensor/metadata/sensor_info
curl -s http://169.254.177.46/api/v1/sensor/config
```

The first proves the sensor is alive and reachable independent of any driver. The second reveals **actual firmware state**, which is how §4 was diagnosed — the launch file and the sensor disagreed, and only the sensor could say so.

### Network layer, in order

```bash
ip -br addr                              # addresses and carrier
ping -c 2 <target>                       # reachability
sudo ethtool <iface> | grep -E 'Speed|Duplex'   # negotiated link
ip -s link show <iface>                  # errors / dropped / missed counters
```

Running all four takes under a minute and eliminated three hypotheses in §3.

### Environment hygiene

```bash
env | grep -i -E 'cyclone|rmw|fastrtps|ros_domain'
grep -n "source\|export" ~/.bashrc | grep -v "^#"
```

The second command found two pieces of stale config that had been silently breaking things.

---

## 8. Current state

### Running

**Spring** (mini PC, `10.0.0.2` / `10.119.16.181` / `169.254.12.36`):
```bash
docker run -it --rm --network host -u root \
  -v ~/ouster_ws:/ouster_ws -w /ouster_ws ouster_robot:working-2026-09-13
cd /ouster_ws && source install/setup.bash
ros2 launch agv_bringup state_publisher_launch.py
```
→ `robot_state_publisher` + `os_driver` (auto-starts, `512x10`, `LEGACY`, 10.002 Hz)

**Manas** (Jetson, `10.0.0.1`):
```bash
ros2 launch zed_wrapper zed_camera.launch.py \
  camera_model:=zed2i camera_name:=zed2i \
  publish_tf:=false publish_map_tf:=false publish_urdf:=true
```
→ `zed2i_state_publisher` + `zed_node` in `zed_container`

### Verified

- Full TF tree resolves identically on both machines
- Clock sync 233 µs (`chronyc tracking`, manas → spring)
- Plain DDS over the LAN cable, no Zenoh, FastDDS both ends, domain 0
- One owner per TF edge, `odom → base_link` deliberately vacant

### Open

| Item | Note |
|---|---|
| **Mount offsets unmeasured** | `0.305 / 0 / 0.08` are inherited placeholders. See §9. |
| `point_type` never retested | Blame was never cleared (§4). Retry `xyzir` alone. |
| `persist_config: False` | Not yet set. Would prevent §4 recurring. |
| `sysctl` not persistent | `rmem_max` set at runtime only; add to `/etc/sysctl.conf` |
| Manas has no committed image | 30+ `Exited` containers, no snapshot |
| `scan_ring` unsuitable | Day-1 §9.3, unchanged |
| Old firmware v2.4.0 | Day-1 §9.4, unchanged |

---

## 9. The one thing that is still wrong

**`cam_x_offset=0.305`, `cam_y_offset=0.00`, `cam_z_offset=0.08` have never been measured.** They are placeholders inherited from before any of this worked.

Every other number in the chain is exact to the millimetre — the 36 mm LiDAR beam offset, the 180° sensor yaw, the 60 mm stereo baseline, the 15 mm camera-center rise. All of it comes free from factory calibration. Those three numbers are the only ones that are yours to get right, and they are currently guesses.

A 3 cm error in camera height misplaces every lane detection projected into the point cloud. It will present as a calibration problem. It is a tape-measure problem.

Same applies to `lidar_z_offset=0.22`. Measure both against the real robot before the next test run.

---

## 10. Housekeeping

```bash
# spring host
docker ps --format '{{.Names}}'
docker commit <name> ouster_robot:lidar-zed-tf-2026-09-14
sudo chown -R spring:spring ~/ouster_ws

# manas host
docker ps --format '{{.Names}}'
docker commit <zed-container> zed_ros2_humble_jetson:tf-working-2026-09-14
docker container prune

# both — make socket buffers survive reboot
echo -e "net.core.rmem_max=26214400\nnet.core.rmem_default=26214400" | sudo tee -a /etc/sysctl.conf
```

Manas has **no committed image at all** and a week of dead containers. Fix that first.

---

## 11. Next: FAST-LIO2

Both inputs are live and correctly framed.

| Requirement | Status |
|---|---|
| Point cloud | `/points` on spring, 10 Hz, `512x10` |
| IMU | `/imu` on spring, `proc_mask` includes `IMU` |
| `os_imu → os_lidar` extrinsic | **free** — Ouster factory calibration |
| `odom → base_link` | vacant, waiting |

**Use the Ouster's IMU, not the ZED's.** It is rigidly coupled to the LiDAR on the same board with a calibrated extrinsic and a shared timebase. The ZED's IMU is on a different rigid body across a LAN cable with no sample-level clock relationship.

**Everything runs on spring.** FAST-LIO2 needs points at 10 Hz and IMU at ~100 Hz in one process. §3 established what happens when that data crosses the cable.

**Before choosing a fork:** check what point fields it requires. Some need per-point timestamps for motion compensation, which `xyzir` does not carry — those want `point_type: native` instead. Decide this before committing to a point type.

**Then set `pos_tracking.publish_tf` correctly forever.** Once FAST-LIO2 owns `odom → base_link`, any other publisher of that edge is a bug.

---

## 12. Repo doc corrections still outstanding

From day 1, not yet applied:

| File | Says | Should say |
|---|---|---|
| `LAN_cable_setup.md` | `10.50.0.1` / `10.50.0.2` | `10.0.0.1` / `10.0.0.2` |
| `zenoh_dds_setup.md` | `-e tcp/10.40.110.86:7447` | stale; Zenoh removed entirely |
| `lidar_bringup.md` | `/ouster_driver` | `/os_driver` |

Stale documentation caused real confusion on both days — the Zenoh IPs on day 1, the `.bashrc` on day 2. A doc that describes a system that no longer exists is worse than no doc, because it makes every diagnosis start from a false premise.
