------------------------------
## Step 1: Force the Physical Mini PC Interface Awake
When an unmanaged Ouster sensor boots into a direct cable link with no router, NetworkManager on the Mini PC sleeps the port because it thinks the wire is dead.
We used the USB-C management backbone to remote-in and force the Mini PC's physical Ethernet card (eno1) awake by creating a dedicated connection profile:

# Force-create a network profile for the LiDAR card
sudo nmcli connection add type ethernet con-name "Ouster_LiDAR" ifname eno1 ipv4.method manual ipv4.addresses 192.168.1.100/24 ipv4.gateway "0.0.0.0"
# Wake the interface up
sudo nmcli connection up "Ouster_LiDAR"

------------------------------
## Step 2: Transition to Link-Local Fallback Mode
Because the LiDAR did not respond to the 192.168.1.x address space, we knew it had dropped down into its built-in safety fallback state (IPv4 Link-Local, which always starts with 169.254.X.Y).
We shifted the Mini PC's interface card onto the exact same frequency layer:

# Shift the method to link-local mapping
sudo nmcli connection modify "Ouster_LiDAR" ipv4.method link-local ipv4.addresses "" ipv4.gateway ""
# Restart the link profile
sudo nmcli connection up "Ouster_LiDAR"

------------------------------
## Step 3: Sniff and Uncover the LiDAR's Real IP
To bridge the connection cleanly inside the container without relying on broken .local DNS tools, we ran a direct network broadcast sweep from the Mini PC terminal line:

ping -b -c 4 -I eno1 169.254.255.255

Once we knocked on the network gateway, we read our local hardware neighborhood cache tables to isolate the true IP signature of the physical sensor:

ping os-122220002210.local

## What Finally Showed the IP (The Golden Match):

PING os-122220002210.local (169.254.177.46) 56(84) bytes of data.
64 bytes from os-122220002210.local (169.254.177.46): icmp_seq=1 ttl=64 time=0.298 ms


* The LiDAR's Real Hidden IP: 169.254.177.46
* The Mini PC's Receiver Interface IP: 169.254.12.36 (from ip a show dev eno1)

------------------------------
## Step 4: Run the Production ROS 2 Driver Inside Docker
With the explicit IP parameters mapped out, we entered your Mini PC container (pc_brain) and launched the driver. By turning off the graphical visualizer (viz:=false), we bypassed the X11/Qt display crashes, keeping your computer brain running as a lightweight embedded master:

# 1. Source your custom workspace files
cd /ouster_ws
source install/setup.bash
# 2. Run the driver with direct IP tracking
ros2 launch ouster_ros sensor.launch.xml \
  sensor_hostname:=169.254.177.46 \
  udp_dest:=169.254.12.36 \
  viz:=false

------------------------------
## Step 5: Initialize the Visual Render on the Jetson Screen
To view your spatial environment, we bypassed the headless constraints of the Mini PC and launched the graphics loops directly on your Jetson's monitor via its high-speed static subnet connection:

# 1. Authorize your local docker system to output windows to your monitor screen
xhost +local:docker
# 2. Spin up your GUI-capable container on the Jetson Host terminal
docker run -it --rm --name jetson_viz --network host --runtime nvidia --gpus all -e DISPLAY=$DISPLAY -v /tmp/.X11-unix:/tmp/.X11-unix -v ~/ros2_ws:/workspace -w /workspace stereolabs/zed:4.1-gl-devel-cuda11.4-ubuntu22.04 bash
# 3. Inside the container terminal, bind the network lane configs
export ROS_DOMAIN_ID=42
export ROS_IP=10.0.0.1
# 4. Fire up the 3D visualization window!
ros2 run rviz2 rviz2

------------------------------

