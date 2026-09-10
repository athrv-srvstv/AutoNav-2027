To Build : 
""


To Run :
"docker run -it --privileged   --runtime nvidia   --network host   -e DISPLAY=$DISPLAY   -v /tmp/.X11-unix:/tmp/.X11-unix   -v /dev:/dev   -v /var/nvidia:/var/nvidia   zed_ros2_humble_jetson-jp6.2.2_sdk5.4.1:latest
"
