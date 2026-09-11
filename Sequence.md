------------------------------
## Phase 1: Host-Level Physical Network Architecture
Because your computers run multiple active network interfaces simultaneously (mobile phone internet tethering, internal computer-to-computer links, and direct sensor lines), the Linux kernel's routing tables can easily get confused. We bypass all automatic IP allocation logic by enforcing fixed, dedicated subnets on the physical host operating systems (outside of Docker).

 ┌─────────────────────────────────┐                 ┌─────────────────────────────────┐
 │         JETSON ORIN NX          │                 │         AOOSTAR MINI PC         │
 │   Host IP Address: 10.0.0.1     │                 │    Host IP Address: 10.0.0.2    │
 └────────────────┬────────────────┘                 └────────┬────────────────┬───────┘
                  │                                           │                │
                  └──────────────[High-Speed LAN Wire]────────┘                │ [LiDAR LAN Wire]
                                 (Static: 10.0.0.x Subnet)                     │ (Link-Local)
                                                                               ▼
                                                                     ┌───────────────────┐
                                                                     │  OUSTER OS1-32    │
                                                                     │   169.254.177.46  │
                                                                     └───────────────────┘

## 1. On the Jetson Orin NX Host Terminal (manas@manas-desktop)
We configure the primary physical LAN interface card (enP8p1s0) to host a permanent static subnet gateway and authorize graphical windows to pass through the container layers.

# 1. Authorize local Docker container layers to draw graphical user interfaces (GUIs) onto your physical monitor screen
xhost +local:docker
# 2. Modify the connection profile to manual static mode, assign the IP, and wipe out conflicting gateway defaults
sudo nmcli connection modify "Wired connection 1" ipv4.method manual ipv4.addresses 10.0.0.1/24 ipv4.gateway "0.0.0.0"
# 3. Force-restart the network adapter interface to apply the static configurations immediately
sudo nmcli connection up "Wired connection 1"
# 4. Explicitly permit high-bandwidth multicast data traffic packet channels down this specific wire interface card
sudo ip link set dev enP8p1s0 multicast on
sudo ip route add 239.255.0.0/16 dev enP8p1s0 2>/dev/null || true

## 2. On the Aoostar Mini PC Host Terminal (spring@spring)
We isolate the Mini PC's two separate physical network hardware cards: enp3s0 (which links straight to the Jetson) and eno1 (which links straight to the Ouster LiDAR sensor).

# 1. Configure the primary computer-to-computer LAN port link onto the matching static subnet array
sudo nmcli connection modify "Jetson Ethernet" ipv4.method manual ipv4.addresses 10.0.0.2/24 ipv4.gateway "0.0.0.0"
sudo nmcli connection up "Jetson Ethernet"
# 2. Open the matching packet pathways on the primary interface card
sudo ip link set dev enp3s0 multicast on
sudo ip route add 239.255.0.0/16 dev enp3s0 2>/dev/null || true
# 3. Allocate a dedicated network manager profile tracking the separate physical LiDAR hardware port card (eno1)
sudo nmcli connection add type ethernet con-name "Ouster_LiDAR" ifname eno1 ipv4.method link-local ipv4.addresses "" ipv4.gateway ""
# 4. Trigger the Ouster port link profile awake into Link-Local safety fallback mode
sudo nmcli connection up "Ouster_LiDAR"


* Why Link-Local for the LiDAR? Out of the box, if an Ouster sensor does not find a router running a DHCP server, it drops into an internal fallback state where it picks its own safe address starting with 169.254.X.Y. Forcing eno1 into link-local mode ensures the Mini PC port drops onto the exact same wavelength to catch it.

------------------------------
## Phase 2: Production Container Deployment
Docker containers are naturally stateless and completely isolated from physical hardware layout properties. To grant them absolute access to your raw network devices, systemic USB busses, and dedicated NVIDIA CUDA graphics pipelines, utilize these highly optimized production run scripts.
## 🖥️ Inside the Mini PC Host Terminal
Launches your master compute workspace with complete host network translation mapping:

docker run -it --rm \
  --name pc_brain \
  --network host \
  -v ~/ros2_ws:/workspace \
  -w /workspace \
  ros:humble \
  bash

## 🟢 Inside the Jetson Orin NX Host Terminal
Launches your camera processing workspace with dedicated GPU hardware passthrough and X11 display display socket mapping:

docker run -it --rm \
  --name jetson_eye \
  --network host \
  --runtime nvidia --gpus all \
  --device /dev/bus/usb \
  -e DISPLAY=$DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v ~/ros2_ws:/workspace \
  -w /workspace \
  stereolabs/zed:4.1-gl-devel-cuda11.4-ubuntu22.04 \
  bash


* Key Parameter Breakdown:
* --network host: Bypasses Docker’s internal isolated firewall routing layers. Nodes inside the container bind directly to the host's physical network connections (10.0.0.x and 169.254.x.x).
   * --runtime nvidia --gpus all: Feeds the Jetson's raw physical graphics card silicon straight into the container layer to execute parallel matrix models without causing system lag.
   * -e DISPLAY=$DISPLAY & -v /tmp/.X11-unix: Maps your physical monitor's display engine into Docker, allowing heavy visual rendering windows (like RViz2) to load smoothly.

------------------------------
## Phase 3: Zenoh-DDS High-Performance Communication Pipeline
Default ROS 2 discovery protocols rely entirely on UDP Multicast, which can frequently get dropped by Docker network walls or physical interface transitions under rough testing terrains.
Zenoh solves this completely. It operates as a transparent, high-performance middleware layer that intercept local ROS 2 topics, packages them into lightweight, reliable TCP frames, and shoots them directly across the wire using an explicit Router-Peer architecture.
## 1. Binary Software Retrieval (Divided to Prevent Terminal Cutoffs)
Execute these blocks one by one inside your running containers to download the correct processor binaries safely.

* Inside the Mini PC Container Terminal (pc_brain - x86 Architecture):

cd /workspace
P1="https://github.com"
P2="/download/1.0.0/zenoh-bridge-dds-1.0.0-x86_64"
P3="-unknown-linux-gnu.tar.gz"
wget "${P1}${P2}${P3}" -O zenoh.tar.gz && tar -xvf zenoh.tar.gz && chmod +x zenoh-bridge-dds

* Inside the Jetson Container Terminal (jetson_eye - ARM64 / aarch64 Architecture):

cd /workspace
P1="https://github.com"
P2="/download/1.0.0/zenoh-bridge-dds-1.0.0-aarch64"
P3="-unknown-linux-gnu.tar.gz"
wget "${P1}${P2}${P3}" -O zenoh.tar.gz && tar -xvf zenoh.tar.gz && chmod +x zenoh-bridge-dds


## 2. Starting the Zenoh TCP Tunnel
Run these commands in their own container terminal panels to link your compute cores securely.

* In the Mini PC Container Panel (Acts as the Listening Router Hub):

./zenoh-bridge-dds --listen tcp/10.0.0.2:7447

* In the Jetson Container Panel (Acts as the Target Client Peer):

./zenoh-bridge-dds --connect tcp/10.0.0.2:7447


The moment you run the Jetson command, both terminals will issue a connection success log. Your direct cross-machine TCP data bridge is now completely live.
------------------------------
## Phase 4: Launching the Sensor Array Packages
To launch your nodes, open a fresh terminal panel inside each running container using docker exec -it <container_name> bash. This keeps your Zenoh tunnels operating undisturbed in the background.
## 1. Waking Up the Ouster OS1-32 LiDAR (On the Mini PC Container)
⚠️ Critical Hardware Requirement: You must place the Ouster LiDAR completely flat/face down on a stable surface during bootup. This allows its internal optical sensors to complete their baseline laser safety calibrations without timing out.

# 1. Bind your local ROS environment metrics explicitly to the LAN connection interface card
export ROS_DOMAIN_ID=42
export ROS_IP=10.0.0.2
# 2. Source your custom spatial package workspace
cd /ouster_ws
source install/setup.bash
# 3. Fire up the node, passing the target sensor parameters and turning off headless visualization (viz:=false)
ros2 launch ouster_ros sensor.launch.xml \
  sensor_hostname:=169.254.177.46 \
  udp_dest:=169.254.12.36 \
  viz:=false


* Why viz:=false? The default launch script tries to open an RViz2 graphics frame natively. Since your Mini PC runs headlessly without a monitor, the graphics driver will crash the node. Disabling it leaves the driver running cleanly as a lightweight background data server.

## 2. Waking Up the ZED 2i Stereo Camera (On the Jetson Container)
⚠️ Critical Hardware Requirement: You must place the ZED 2i camera completely flat/face down on a vibration-free surface during bootup. This allows its highly sensitive internal Inertial Measurement Unit (IMU) gyroscopes to complete their structural gravity-vector calculations without locking up.

# 1. Bind your matching ROS environment metrics to the Jetson network card
export ROS_DOMAIN_ID=42
export ROS_IP=10.0.0.1
# 2. Source your local camera wrapper installation files
cd /workspace
source install/setup.bash
# 3. Launch the native ZED component drivers utilizing its specialized model tracking matrix
ros2 launch zed_wrapper zed_camera.launch.py model:=zed2i

------------------------------
## Phase 5: Frame Fusion Coordinate Alignment & Verification
Now that both sensors are streaming data over your distributed network layer simultaneously, you must stitch their spatial coordinate frames together. Without a transformation node, mapping engines (like RViz2) cannot determine where the camera sits relative to the LiDAR, and will refuse to draw the pixels.
## Step 1: Initialize the Spatial Bridge & Visualizer (On the Jetson Container)
Open two separate terminal panels inside your jetson_eye container and execute:

# Panel A: Publish a static 10cm forward transform offset linking your LiDAR frame directly to your Camera frame
ros2 run tf2_ros static_transform_publisher "0.1" "0" "0" "0" "0" "0" "os_sensor" "zed_left_camera_frame"

# Panel B: Fire up the main 3D visualization user dashboard window
export ROS_DOMAIN_ID=42
export ROS_IP=10.0.0.1
ros2 run rviz2 rviz2

## Step 2: Configure the Master Verification Panel in RViz2
Once the graphical RViz2 application loads onto your Jetson monitor screen, configure these properties in the left-hand configuration tree:

   1. Fixed Frame Update: Look under Global Options -> Click Fixed Frame -> Change the text string from map to exactly: os_sensor.
   2. Inject the LiDAR Layer: Click Add (bottom left) -> By Topic Tab -> Select /ouster/points -> Choose PointCloud2.
   3. Inject the Camera Layer: Click Add -> By Topic Tab -> Select /zed/zed_node/point_cloud/cloud_registered -> Choose PointCloud2.
   4. Optimize Rendering: Expand the configuration options for both point cloud layers on the left side menu panel:
   * Change Style from Points to Boxes.
      * Increase Size (m) to 0.03.
   



