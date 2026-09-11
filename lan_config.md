No need to apologize at all! It is incredibly smart to document this. When you power down the robot and turn it back on tomorrow, or if a cable gets unplugged on the field, NetworkManager will try to reset these interfaces.
Here is the exact, step-by-step Network Recovery Procedure you can follow to restore your high-speed LAN connection from a completely blank slate.
------------------------------
## Phase 1: Emergency Access (Bypassing the Monitor Swapping)
If the main LAN cable is asleep and you cannot SSH over it, do not swap your monitor cables. Use the USB A-to-C cable as a temporary remote control bridge.

   1. Connect the USB-C end into the Jetson’s OTG/Service port, and the USB-A end into the Mini PC.
   2. Open a local terminal on your Jetson screen (manas@manas-desktop) and run:
   
   ssh spring@192.168.55.100
   
   3. Type your Mini PC password. You are now controlling both PCs from one monitor.

------------------------------
## Phase 2: Restoring the High-Speed Static LAN Configuration
Now that you have terminal windows open for both systems, apply the static 10.0.0.x subnets to wake up the main Ethernet wire line.
## Step 1: Configure the Jetson Orin NX (enP8p1s0)
Open a terminal tab running locally on your Jetson and run this single-line command to force the interface out of automatic mode:

sudo nmcli connection modify "Wired connection 1" ipv4.method manual ipv4.addresses 10.0.0.1/24 ipv4.gateway "0.0.0.0"
sudo nmcli connection up "Wired connection 1"

## Step 2: Configure the Aoostar Mini PC (enp3s0)
Switch over to your Mini PC SSH tab (spring@spring) and run the corresponding configuration command:

sudo nmcli connection modify "Jetson Ethernet" ipv4.method manual ipv4.addresses 10.0.0.2/24 ipv4.gateway "0.0.0.0"

(Note: If the profile is named differently after a reboot, run nmcli connection show to check the exact label, then turn it on via: sudo nmcli connection up "Your_Profile_Name").
------------------------------
## Phase 3: Hardware Verification Check
Before launching any Docker layers, confirm that the physical copper wire is actively transmitting data between the two platforms.

   1. From your Jetson terminal tab, ping the Mini PC's static address directly:
   
   ping -c 4 10.0.0.2
   
   2. Check for 0% packet loss and sub-millisecond return latencies (< 1.00 ms). If it succeeds, the connection is solid.

------------------------------
## Phase 4: Initializing the Isolated Docker Containers
With the hardware channels operating smoothly, boot up your production workspaces.

   1. On the Mini PC Terminal Window (spring):
   
   docker run -it --rm --name pc_brain --network host -v ~/ros2_ws:/workspace -w /workspace ros:humble bash
   
   2. On the Jetson Terminal Window (manas):
   
   docker run -it --rm --name jetson_eye --network host --runtime nvidia --gpus all --device /dev/bus/usb -v ~/ros2_ws:/workspace -w /workspace stereolabs/zed:4.1-gl-devel-cuda11.4-ubuntu22.04 bash
   
   
------------------------------
## Phase 5: Priming the Container Network Environments
Because multiple networks are active simultaneously (e.g., your phone tethering and your internal LAN cross-link), you must explicitly isolate the ROS 2 DDS execution channels inside your active containers.

* Inside the Mini PC Container Prompt (pc_brain):

export ROS_DOMAIN_ID=42
export ROS_IP=10.0.0.2

* Inside the Jetson Container Prompt (jetson_eye):

export ROS_DOMAIN_ID=42
export ROS_IP=10.0.0.1


------------------------------
## Phase 6: Running the Custom Node Data Test

   1. On the Jetson Container Tab: Run your custom talker/publisher node:
   
   source install/setup.bash
   ros2 run <your_talker_package> <your_talker_node>
   
   2. On the Mini PC Container Tab: Verify the network discovery layer before running your scripts:
   
   ros2 topic list
   
   
If your custom topic paths appear cleanly inside the topic list, your distributed system is fully operational and ready to process sensor pipelines!
Now that our networking playbook is complete, what sensor should we configure next? Let me know:

* What is the exact model of your Ouster LiDAR (e.g., OS1-32 or OS1-64)?
* Do you want to structure the LiDAR driver execution scripts on the Mini PC, or configure the ZED 2i tf matrix parameters on the Jetson?


