
 # ROS 2 Multi-Machine Communication over Direct Ethernet LAN using Zenoh

 ## 1\. Objective

 Set up two Ubuntu machines so that:

 - SSH works between the machines over a normal Ethernet/LAN cable.
- ROS 2 nodes on both machines can communicate.
- Zenoh provides the transport between the two ROS 2 systems.
- The setup does not depend on the previous USB networking connection.
- The Ethernet connection uses static IP addresses.
- Zenoh runs inside Docker containers using host networking.

 ### Final topology

```
                    Direct Ethernet Cable
                 10.50.0.0/24 network
                         │
          ┌──────────────┴──────────────┐
          │                             │
          ▼                             ▼
┌────────────────────┐         ┌────────────────────┐
│ MANAS              │         │ SPRING             │
│                    │         │                    │
│ enP8p1s0            │         │ enp3s0             │
│ 10.50.0.1/24       │         │ 10.50.0.2/24       │
│                    │         │                    │
│ Zenoh Router       │◄───────►│ Zenoh Bridge       │
│ TCP :7447          │         │                    │
│                    │         │ ROS 2              │
│ ROS 2              │         │                    │
└────────────────────┘         └────────────────────┘
```

---

 # 2\. Machines

 ## Manas machine

 Ethernet interface:

```
enP8p1s0
```

 Final Ethernet address:

```
10.50.0.1/24
```

 ## Spring machine

 Ethernet interface:

```
enp3s0
```

 Final Ethernet address:

```
10.50.0.2/24
```

 Spring is running:

```
Ubuntu 22.04.5 LTS
```

 The ROS 2 environment used for the Zenoh bridge is:

```
ROS_DISTRO=humble
```

---

 # 3\. Initial Ethernet diagnosis

 On Manas, the Ethernet interface initially appeared as:

```
enP8p1s0: <NO-CARRIER,BROADCAST,MULTICAST,UP>
```

 This indicated that the Ethernet interface existed but the physical link was not detected.

 After connecting the normal LAN cable, the interface changed to:

```
enP8p1s0: <BROADCAST,MULTICAST,UP,LOWER_UP>
```

 The important field is:

```
LOWER_UP
```

 This confirms that the physical Ethernet link is established.

 The interface was explicitly brought up with:

```
sudo ip link set enP8p1s0 up
```

---

 # 4\. Identifying the existing network addresses

 Manas initially had an address on the USB networking interface:

```
usb2
10.40.110.184/24
```

 It also had:

```
l4tbr0
192.168.55.1/24
```

 These were associated with the previous USB networking setup.

 The goal was to avoid depending on those interfaces and establish a separate Ethernet network.

 Therefore, a dedicated private subnet was chosen:

```
10.50.0.0/24
```

 with:

```
Manas  = 10.50.0.1
Spring = 10.50.0.2
```

---

 # 5\. Configure Manas Ethernet IP

 On Manas:

```
sudo ip link set enP8p1s0 up
```

 Then assign:

```
sudo ip addr add 10.50.0.1/24 dev enP8p1s0
```

 Verify:

```
ip addr show enP8p1s0
```

 Expected result:

```
enP8p1s0: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 10.50.0.1/24
```

---

 # 6\. Diagnose Spring Ethernet interface

 On Spring:

```
ip -br addr
```

 Initially, the Ethernet interface was:

```
enp3s0    UP    169.254.50.186/16
```

 The `169.254.x.x` address is a link-local address. It indicated that the interface was physically connected but did not have the desired address from DHCP.

 Spring's relevant interfaces were:

```
wlp4s0       10.72.0.162/21
enp3s0       169.254.50.186/16
```

 The Wi-Fi interface was left untouched.

---

 # 7\. Temporarily configure Spring Ethernet IP

 Initially, the address was attempted using:

```
sudo ip addr flush dev enp3s0
sudo ip addr add 10.50.0.2/24 dev enp3s0
sudo ip link set enp3s0 up
```

 However, NetworkManager subsequently restored the link-local address.

 This was diagnosed with:

```
nmcli device status
```

 The important result was:

```
enp3s0    ethernet    connected    jetson
```

 Then:

```
nmcli connection show
```

 showed:

```
jetson    ...    ethernet    enp3s0
```

 Therefore, the Ethernet interface was being managed by NetworkManager through the connection named:

```
jetson
```

---

 # 8\. Configure Spring Ethernet IP persistently

 The NetworkManager connection was changed to use a static IPv4 address.

 On Spring:

```
sudo nmcli connection modify "jetson" \
  ipv4.method manual \
  ipv4.addresses 10.50.0.2/24
```

 Then activate the connection:

```
sudo nmcli connection up "jetson"
```

 Expected result:

```
Connection successfully activated
```

---

 # 9\. Verify Spring Ethernet configuration

 Check:

```
ip -br addr show enp3s0
```

 Expected:

```
enp3s0    UP    10.50.0.2/24
```

 Then verify the persistent NetworkManager configuration:

```
nmcli connection show "jetson" | grep -E 'ipv4.method|ipv4.addresses'
```

 Expected:

```
ipv4.method:    manual
ipv4.addresses: 10.50.0.2/24
```

 This confirmed that the address was configured persistently through NetworkManager.

---

 # 10\. Test Ethernet connectivity

 From Manas:

```
ping -c 4 10.50.0.2
```

 Successful result:

```
4 packets transmitted, 4 received, 0% packet loss
```

 Example observed latency:

```
rtt min/avg/max/mdev = 0.282/0.664/1.267/0.403 ms
```

 This confirmed that the two machines could communicate directly over Ethernet.

---

 # 11\. Test SSH over Ethernet

 From Manas:

```
ssh spring@10.50.0.2
```

 The first connection produced the normal SSH host-key warning:

```
The authenticity of host '10.50.0.2' can't be established.
```

 After accepting the key, SSH successfully connected.

 The Spring login showed:

```
Last login: ... from 10.50.0.1
```

 This is important because it confirms that SSH was coming from the new Ethernet address:

```
10.50.0.1
```

 rather than the old USB networking address.

 The final SSH command is therefore:

```
ssh spring@10.50.0.2
```

---

 # 12\. Verify routing

 From Manas, the Ethernet route can be verified with:

```
ip route get 10.50.0.2
```

 Expected output should contain:

```
dev enP8p1s0
src 10.50.0.1
```

 This confirms that traffic destined for Spring's Ethernet address is routed through:

```
enP8p1s0
```

---

 # 13\. Zenoh setup

 Zenoh was used to bridge ROS 2 communication between the two machines.

 The Docker image used was:

```
eclipse/zenoh-bridge-ros2dds:latest
```

 The version observed during setup was:

```
zenoh-bridge-ros2dds v1.10.1
```

 Docker was run with host networking:

```
--net=host
```

 This allows the container to directly use the host's network interfaces and ports.

---

 # 14\. Start Zenoh on Manas

 On Manas:

```
docker run --rm -it --net=host \
  -e ROS_DISTRO=humble \
  eclipse/zenoh-bridge-ros2dds:latest
```

 The Zenoh router listens on:

```
TCP 7447
```

 The important network endpoint is:

```
10.50.0.1:7447
```

 Leave this container running.

---

 # 15\. Start Zenoh on Spring

 SSH into Spring over Ethernet:

```
ssh spring@10.50.0.2
```

 Then start the bridge:

```
docker run --rm -it --net=host \
  -e ROS_DISTRO=humble \
  eclipse/zenoh-bridge-ros2dds:latest \
  -e tcp/10.50.0.1:7447
```

 The important argument is:

```
-e tcp/10.50.0.1:7447
```

 This instructs the Spring Zenoh instance to connect to the Zenoh router running on Manas.

---

 # 16\. Zenoh connection test

 The Zenoh connection was successfully established using:

```
tcp/10.50.0.1:7447
```

 This confirmed that Zenoh traffic was able to travel over the Ethernet network:

```
Spring
10.50.0.2
     │
     │ Ethernet
     ▼
Manas
10.50.0.1:7447
```

---

 # 17\. Important Zenoh debugging issue

 During the earlier setup, starting two Zenoh routers on the same host caused:

```
Unable to open listener tcp/[::]:7447
```

 and:

```
Address in use (os error 98)
```

 This occurred because TCP port `7447` was already occupied.

 The solution was not to start another local router on the same port. Instead, the second machine should connect to the existing Zenoh router using:

```
-e tcp/10.50.0.1:7447
```

 Therefore:

 - Manas acts as the Zenoh router.
- Spring connects to Manas.
- Both ROS 2 systems are bridged through that Zenoh connection.

---

 # 18\. ROS 2 environment

 The Zenoh containers were started with:

```
-e ROS_DISTRO=humble
```

 This avoids the warning encountered earlier when `ROS_DISTRO` was not defined.

 Without the variable, Zenoh reported:

```
ROS_DISTRO environment variable is not set.
Assuming 'iron'
```

 Therefore, the final commands explicitly use:

```
-e ROS_DISTRO=humble
```

---

 # 19\. ROS 2 testing

 Initially, the following command failed:

```
ros2 run demo_nodes_cpp talker
```

 with:

```
Package 'demo_nodes_cpp' not found
```

 This means the ROS 2 demo package was not installed in that ROS environment.

 If needed, install it with:

```
sudo apt update
sudo apt install ros-humble-demo-nodes-cpp
```

 Then source ROS 2:

```
source /opt/ros/humble/setup.bash
```

 Start a talker on one machine:

```
ros2 run demo_nodes_cpp talker
```

 Start a listener on the other:

```
ros2 run demo_nodes_cpp listener
```

 Successful listener output looks like:

```
I heard: [Hello World: ...]
```

 This demonstrates actual ROS 2 message communication between the machines.

---

 # 20\. Final working commands

 ## Manas

 Start the Zenoh router:

```
docker run --rm -it --net=host \
  -e ROS_DISTRO=humble \
  eclipse/zenoh-bridge-ros2dds:latest
```

 Manas Ethernet:

```
Interface: enP8p1s0
IP:        10.50.0.1/24
```

---

 ## Spring

 Persistent Ethernet configuration:

```
NetworkManager connection: jetson
Interface:                enp3s0
IP:                        10.50.0.2/24
```

 SSH from Manas:

```
ssh spring@10.50.0.2
```

 Start Zenoh:

```
docker run --rm -it --net=host \
  -e ROS_DISTRO=humble \
  eclipse/zenoh-bridge-ros2dds:latest \
  -e tcp/10.50.0.1:7447
```

---

 # 21\. Final verification checklist

 The following tests were successfully completed.

 ### Physical Ethernet

```
ip link show enP8p1s0
```

 Manas showed:

```
LOWER_UP
```

 Spring showed:

```
LOWER_UP
```

 ### IP addressing

 Manas:

```
10.50.0.1/24
```

 Spring:

```
10.50.0.2/24
```

 ### Ping

```
ping -c 4 10.50.0.2
```

 Result:

```
4 packets transmitted
4 received
0% packet loss
```

 ### SSH

```
ssh spring@10.50.0.2
```

 Successfully connected.

 Spring reported:

```
Last login: ... from 10.50.0.1
```

 ### Zenoh

 Spring successfully connected to:

```
tcp/10.50.0.1:7447
```

 ### ROS 2

 ROS 2 talker/listener communication was successfully tested through the Zenoh bridge.

---

 # 22\. USB networking vs final Ethernet networking

 The earlier setup used USB networking interfaces such as:

```
usb2
10.40.110.184
```

 and:

```
192.168.55.x
```

 Those were useful during the initial setup but are not required for the final LAN architecture.

 The final communication path is:

```
ROS 2
  │
  ▼
zenoh-bridge-ros2dds
  │
  ▼
Zenoh TCP :7447
  │
  ▼
enP8p1s0 / enp3s0
  │
  ▼
Physical Ethernet cable
  │
  ▼
enp3s0 / enP8p1s0
  │
  ▼
Zenoh
  │
  ▼
zenoh-bridge-ros2dds
  │
  ▼
ROS 2
```

 SSH uses the same Ethernet network:

```
ssh spring@10.50.0.2
```

---

 # 23\. Final architecture

```
                         DIRECT LAN CABLE
                    ┌────────────────────────┐
                    │                        │
                    │      10.50.0.0/24     │
                    │                        │
                    └────────────────────────┘
                              │
             ┌────────────────┴────────────────┐
             │                                 │
             ▼                                 ▼

       ┌───────────────┐                 ┌───────────────┐
       │     MANAS     │                 │    SPRING     │
       │               │                 │               │
       │ enP8p1s0      │                 │ enp3s0        │
       │ 10.50.0.1     │                 │ 10.50.0.2     │
       │               │                 │               │
       │ Zenoh Router  │◄── TCP :7447 ──►│ Zenoh Bridge  │
       │               │                 │               │
       │ ROS 2         │◄── ROS topics ─►│ ROS 2         │
       │               │                 │               │
       └───────────────┘                 └───────────────┘
              │                                  │
              │                                  │
              ▼                                  ▼
          SSH server                         SSH client
```

 # 24\. Troubleshooting reference

 ## Ethernet interface says `NO-CARRIER`

 Example:

```
enP8p1s0: <NO-CARRIER,...>
```

 Check:

```
ip link show enP8p1s0
```

 A physical cable connection should eventually produce:

```
LOWER_UP
```

 Also verify the cable and the Ethernet port on the other machine.

---

 ## Spring gets a `169.254.x.x` address

 Example:

```
enp3s0    UP    169.254.50.186/16
```

 This is a link-local address.

 Check NetworkManager:

```
nmcli device status
nmcli connection show
```

 Identify the connection associated with `enp3s0`.

 In this setup it was:

```
jetson
```

 Configure it using:

```
sudo nmcli connection modify "jetson" \
  ipv4.method manual \
  ipv4.addresses 10.50.0.2/24
```

 Then:

```
sudo nmcli connection up "jetson"
```

---

 ## Ping fails

 Check both addresses:

 Manas:

```
ip addr show enP8p1s0
```

 Spring:

```
ip addr show enp3s0
```

 Expected:

```
Manas:  10.50.0.1/24
Spring: 10.50.0.2/24
```

 Then:

```
ping -c 4 10.50.0.2
```

---

 ## SSH fails

 Check that Spring is reachable:

```
ping -c 4 10.50.0.2
```

 Then:

```
ssh spring@10.50.0.2
```

 Verify routing:

```
ip route get 10.50.0.2
```

 Expected:

```
dev enP8p1s0
src 10.50.0.1
```

---

 ## Zenoh reports `Address in use`

 If you see:

```
Address in use (os error 98)
```

 another Zenoh process is already using port `7447`.

 Check:

```
sudo ss -ltnp | grep 7447
```

 Do not start another router on the same host/port.

 The intended architecture is:

```
Manas → Zenoh router
Spring → connect to tcp/10.50.0.1:7447
```

---

 ## ROS 2 says `demo_nodes_cpp` is missing

 Install:

```
sudo apt update
sudo apt install ros-humble-demo-nodes-cpp
```

 Then:

```
source /opt/ros/humble/setup.bash
```

---

 # 25\. Quick-start procedure

 For future use, the minimal procedure is:

 ### Manas

 Ensure Ethernet is configured:

```
enP8p1s0 = 10.50.0.1/24
```

 Start Zenoh:

```
docker run --rm -it --net=host \
  -e ROS_DISTRO=humble \
  eclipse/zenoh-bridge-ros2dds:latest
```

 ### Spring

 Ensure NetworkManager has:

```
jetson
enp3s0
10.50.0.2/24
```

 Test:

```
ping -c 4 10.50.0.1
```

 SSH:

```
ssh spring@10.50.0.2
```

 Start Zenoh:

```
docker run --rm -it --net=host \
  -e ROS_DISTRO=humble \
  eclipse/zenoh-bridge-ros2dds:latest \
  -e tcp/10.50.0.1:7447
```

 Then run the ROS 2 nodes.

---

 # 26\. Final result

 The final system provides:

```
                    NORMAL ETHERNET CABLE
                           │
        ┌──────────────────┴──────────────────┐
        │                                     │
     MANAS                                  SPRING
  10.50.0.1                               10.50.0.2
        │                                     │
        │────────────── SSH ─────────────────│
        │                                     │
        │──────────── Zenoh :7447 ───────────│
        │                                     │
        │────────────── ROS 2 ───────────────│
```

 The setup is therefore independent of the temporary USB networking connection and provides a dedicated Ethernet network for **SSH + Zenoh + ROS 2 inter-machine communication**.
