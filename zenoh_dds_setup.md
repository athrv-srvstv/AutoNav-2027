# ROS 2 ↔ ROS 2 Communication Over Zenoh Using Docker

 ## 1\. Objective

 This document describes the working setup used to communicate between two ROS 2 systems running on separate machines over a network using **Eclipse Zenoh**.

 The setup uses:

 - Ubuntu 22.04.5 LTS
- ARM64 / `aarch64`
- ROS 2 Humble
- Docker
- `eclipse/zenoh-bridge-ros2dds:latest`
- Zenoh bridge version `1.10.1`
- TCP communication between the two machines
- Docker host networking (`--net=host`)

 The resulting communication path is:

```
Machine A                              Machine B

ROS 2 nodes                            ROS 2 nodes
    │                                      │
    │ DDS                                  │ DDS
    ▼                                      ▼
zenoh-bridge-ros2dds                zenoh-bridge-ros2dds
    │                                      │
    └────────────── TCP / Zenoh ───────────┘
                   port 7447
```

---

 # 2\. Machine Configuration

 ## Machine A

 Machine A:

```
OS: Ubuntu 22.04.5 LTS
Architecture: aarch64 / ARM64
IP: 10.40.110.86
```

 Zenoh endpoint:

```
tcp/10.40.110.86:7447
```

 Machine A acts as the Zenoh router/listener.

 ## Machine B

 Machine B:

```
IP: 10.40.110.105
```

 Machine B connects to Machine A over:

```
tcp/10.40.110.86:7447
```

 Both systems use ROS 2 domain `0`.

---

 # 3\. Why `zenoh-bridge-ros2dds` Was Used

 For ROS 2 communication, use:

```
zenoh-bridge-ros2dds
```

 rather than the generic:

```
zenoh-bridge-dds
```

 The ROS 2-specific bridge provides better integration with the ROS 2 graph and ROS 2 tooling, including topics, services, actions, and commands such as `ros2 topic list`. The official Zenoh documentation recommends `zenoh-bridge-ros2dds` for ROS 2 applications.  GitHub+1

---

 # 4\. Check System Architecture

 Before installing Zenoh, check the machine architecture:

```
uname -m
```

 For this setup, the result was:

```
aarch64
```

 This means the machine is ARM64.

 This is important because an `x86_64` Zenoh binary should not be downloaded for this machine.

 The official Docker image supports both `amd64` and `arm64`, so using the Docker image avoids manually selecting the architecture-specific binary.  GitHub

---

 # 5\. Check Ubuntu Version

 Check the operating system:

```
cat /etc/os-release
```

 Expected:

```
PRETTY_NAME="Ubuntu 22.04.5 LTS"
NAME="Ubuntu"
VERSION_ID="22.04"
VERSION="22.04.5 LTS (Jammy Jellyfish)"
VERSION_CODENAME=jammy
```

---

 # 6\. Install Docker

 If Docker is not already installed, install Docker before pulling the Zenoh image.

 First update the package index:

```
sudo apt update
```

 Install the required packages:

```
sudo apt install -y ca-certificates curl
```

 Install Docker using the official Docker installation method appropriate for the system.

 After installation, verify:

```
docker --version
```

 Also verify that Docker is working:

```
sudo docker run hello-world
```

 If the `docker` command requires `sudo`, either continue using `sudo docker ...` or configure the current user to use Docker without `sudo`.

---

 # 7\. Download / Install the Zenoh Docker Image

 The Zenoh ROS 2 DDS bridge is distributed as an official Docker image.

 Pull the latest image:

```
docker pull eclipse/zenoh-bridge-ros2dds:latest
```

 The official Zenoh documentation lists this as the Docker installation command and states that the image is available for both amd64 and arm64.  GitHub

 Verify that the image exists:

```
docker images | grep zenoh
```

 You should see an entry similar to:

```
eclipse/zenoh-bridge-ros2dds
```

---

 # 8\. Verify the Zenoh Bridge Version

 The image used in this setup reported:

```
zenoh-bridge-ros2dds v1.10.1
```

 You can start the image to verify the version:

```
docker run --rm -it \
  eclipse/zenoh-bridge-ros2dds:latest \
  --help
```

 Alternatively, start the bridge using the commands in the following sections.

---

 # 9\. Docker Networking

 The bridge must use host networking:

```
--net=host
```

 This allows the Zenoh bridge container to directly use the host's network interfaces and expose TCP port `7447` directly on the host.

 For ROS 2/DDS communication, this is particularly important because DDS uses UDP multicast. The Zenoh documentation specifically recommends host networking for this Docker deployment.  GitHub

---

 # 10\. ROS\_DISTRO Configuration

 The bridge initially produced this warning:

```
ROS_DISTRO environment variable is not set.
Assuming 'iron'
```

 Since the ROS 2 system is Humble, explicitly pass:

```
-e ROS_DISTRO=humble
```

 Therefore, the final Docker commands use:

```
-e ROS_DISTRO=humble
```

---

 # 11\. Machine A — Start Zenoh Bridge

 Machine A acts as the Zenoh router.

 Run:

```
docker run --rm -it --net=host \
  -e ROS_DISTRO=humble \
  eclipse/zenoh-bridge-ros2dds:latest
```

 The bridge starts in router mode and listens for incoming Zenoh connections on TCP port `7447`.

 The successful startup produced:

```
zenoh-bridge-ros2dds v1.10.1
Successfully started plugin ros2dds
```

 and:

```
Zenoh can be reached at:
tcp/10.40.110.86:7447
```

 Keep this process running.

---

 # 12\. Machine B — Connect to Machine A

 Machine B connects to the Zenoh bridge on Machine A.

 Run:

```
docker run --rm -it --net=host \
  -e ROS_DISTRO=humble \
  eclipse/zenoh-bridge-ros2dds:latest \
  -e tcp/10.40.110.86:7447
```

 The important argument is:

```
-e tcp/10.40.110.86:7447
```

 This tells Machine B's Zenoh bridge to connect to Machine A.

 The official Zenoh documentation describes this deployment pattern as one bridge listening on the robot/host and the other bridge connecting using `-e tcp/<robot-ip>:7447`.  GitHub

 Machine B's own Zenoh endpoint was:

```
tcp/10.40.110.105:7447
```

---

 # 13\. Network Connectivity Test

 Before starting the bridge on Machine B, verify that Machine B can reach Machine A.

 From Machine B:

```
ping 10.40.110.86
```

 Then test TCP port `7447`:

```
nc -vz 10.40.110.86 7447
```

 If `nc` is not installed:

```
sudo apt update
sudo apt install -y netcat-openbsd
```

 Then:

```
nc -vz 10.40.110.86 7447
```

 Expected result:

```
Connection to 10.40.110.86 7447 port [tcp/*] succeeded!
```

---

 # 14\. ROS 2 Environment

 Both machines use ROS 2 Humble.

 Source ROS 2:

```
source /opt/ros/humble/setup.bash
```

 Check the ROS distribution:

```
echo $ROS_DISTRO
```

 Expected:

```
humble
```

 Check the ROS domain:

```
echo $ROS_DOMAIN_ID
```

 Expected for this setup:

```
0
```

 If it is not set, ROS 2 uses domain `0` by default.

---

 # 15\. Install ROS 2 Test Nodes

 The `demo_nodes_cpp` package is useful for testing communication.

 Install it inside the ROS 2 environment:

```
sudo apt update
sudo apt install -y ros-humble-demo-nodes-cpp
```

 If running as `root` inside a container:

```
apt update
apt install -y ros-humble-demo-nodes-cpp
```

 Source ROS 2:

```
source /opt/ros/humble/setup.bash
```

 Verify:

```
ros2 pkg list | grep demo_nodes_cpp
```

 Expected:

```
demo_nodes_cpp
```

---

 # 16\. Test ROS 2 Communication

 ## Machine A — Talker

 With the Zenoh bridge still running, open another terminal.

 Start the ROS 2 talker:

```
ros2 run demo_nodes_cpp talker
```

 Expected output:

```
Publishing: 'Hello World: 0'
Publishing: 'Hello World: 1'
Publishing: 'Hello World: 2'
...
```

---

 ## Machine B — Listener

 On Machine B, open another terminal.

 Check the topics:

```
ros2 topic list
```

 The `/chatter` topic should appear.

 Then run:

```
ros2 run demo_nodes_cpp listener
```

 Expected:

```
I heard: [Hello World: 0]
I heard: [Hello World: 1]
I heard: [Hello World: 2]
...
```

 This confirms end-to-end ROS 2 communication.

---

 # 17\. Final Communication Architecture

 The final communication path is:

```
                    Network
               10.40.110.x
                     │
        ┌────────────┴────────────┐
        │                         │
        ▼                         ▼
   MACHINE A                 MACHINE B
   10.40.110.86              10.40.110.105
        │                         │
   ROS 2 Humble              ROS 2 Humble
        │                         │
       DDS                       DDS
        │                         │
        ▼                         ▼
Zenoh Bridge A              Zenoh Bridge B
        │                         │
        │                         │
        └────── TCP :7447 ────────┘
                    │
                  Zenoh
```

---

 # 18\. Final Commands — Quick Reference

 ## Machine A

 ### Install/pull Zenoh

```
docker pull eclipse/zenoh-bridge-ros2dds:latest
```

 ### Run Zenoh bridge

```
docker run --rm -it --net=host \
  -e ROS_DISTRO=humble \
  eclipse/zenoh-bridge-ros2dds:latest
```

 Machine A:

```
10.40.110.86:7447
```

---

 ## Machine B

 ### Install/pull Zenoh

```
docker pull eclipse/zenoh-bridge-ros2dds:latest
```

 ### Run Zenoh bridge

```
docker run --rm -it --net=host \
  -e ROS_DISTRO=humble \
  eclipse/zenoh-bridge-ros2dds:latest \
  -e tcp/10.40.110.86:7447
```

 Machine B:

```
10.40.110.105:7447
```

---

 # 19\. ROS 2 Test Commands

 Install the test package:

```
apt update
apt install -y ros-humble-demo-nodes-cpp
```

 Source ROS 2:

```
source /opt/ros/humble/setup.bash
```

 Check:

```
echo $ROS_DISTRO
echo $ROS_DOMAIN_ID
```

 Expected:

```
humble
0
```

 Start the publisher on Machine A:

```
ros2 run demo_nodes_cpp talker
```

 Start the subscriber on Machine B:

```
ros2 run demo_nodes_cpp listener
```

 Check topics:

```
ros2 topic list
```

---

 # 20\. Troubleshooting

 ## Error: `404 Not Found`

 An incorrect GitHub release URL was initially constructed for `zenoh-bridge-dds`.

 The setup was changed to use the official Docker image:

```
docker pull eclipse/zenoh-bridge-ros2dds:latest
```

 This also avoided manually selecting the ARM64 binary.

---

 ## Error: Wrong Architecture

 The machine reported:

```
uname -m
```

 as:

```
aarch64
```

 Therefore, an `x86_64` binary should not be used.

 The official Docker image supports both amd64 and arm64.  GitHub

---

 ## Error: `Address in use`

 If the bridge reports:

```
Address in use (os error 98)
```

 or:

```
Unable to open listener tcp/[::]:7447
```

 another Zenoh process is already using port `7447`.

 Do not start two Zenoh routers on the same host using the same port.

 For this setup:

 - Machine A runs the listening/router bridge.
- Machine B connects to Machine A using `-e tcp/10.40.110.86:7447`.

---

 ## Warning: `ROS_DISTRO environment variable is not set`

 If the bridge reports:

```
ROS_DISTRO environment variable is not set.
Assuming 'iron'
```

 start it with:

```
-e ROS_DISTRO=humble
```

 For example:

```
docker run --rm -it --net=host \
  -e ROS_DISTRO=humble \
  eclipse/zenoh-bridge-ros2dds:latest
```

---

 ## Warning: `ParticipantEntitiesInfo`

 A warning was observed on Machine B:

```
Error receiving ParticipantEntitiesInfo on ros_discovery_info:
failed to fill whole buffer
```

 This occurred during ROS 2 discovery processing.

 The bridge itself successfully started and loaded the ROS 2 DDS plugin.

 First test actual ROS 2 communication using:

```
ros2 topic list
```

 and:

```
ros2 run demo_nodes_cpp talker
```

 followed by:

```
ros2 run demo_nodes_cpp listener
```

 before changing the Zenoh configuration.

---

 # 21\. Important DDS Configuration Note

 The Zenoh documentation warns that DDS communication should not simultaneously occur directly between the two hosts being bridged, because duplicate or looping traffic can occur. The recommended approaches include isolating DDS discovery/traffic or using different ROS domain IDs.  GitHub

 For the initial working test, both systems used:

```
ROS_DOMAIN_ID=0
```

 If the two machines are on a network where normal DDS multicast allows the ROS 2 nodes to directly discover each other, additional DDS isolation may be required for a production deployment.

---

 # 22\. Complete Setup From Scratch

 For a new ARM64 Ubuntu 22.04 machine:

```
# Check architecture
uname -m

# Check Ubuntu version
cat /etc/os-release

# Verify Docker
docker --version

# Pull Zenoh ROS 2 bridge
docker pull eclipse/zenoh-bridge-ros2dds:latest
```

 On Machine A:

```
docker run --rm -it --net=host \
  -e ROS_DISTRO=humble \
  eclipse/zenoh-bridge-ros2dds:latest
```

 On Machine B:

```
docker run --rm -it --net=host \
  -e ROS_DISTRO=humble \
  eclipse/zenoh-bridge-ros2dds:latest \
  -e tcp/10.40.110.86:7447
```

 Then test:

```
# Machine A
ros2 run demo_nodes_cpp talker
```

 and:

```
# Machine B
ros2 run demo_nodes_cpp listener
```

 If the listener receives the talker's messages, ROS 2 communication across the two machines through Zenoh is working.

 One correction worth emphasizing in the documentation: **`docker pull` is the actual Zenoh installation/download step for this approach**; `docker run` then creates and starts the bridge container. The official Zenoh docs explicitly document that Docker image workflow.  GitHub

  Sources
