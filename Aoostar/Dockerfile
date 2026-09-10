FROM ros:humble-ros-base

ARG USERNAME=robot
ARG USER_UID=1000
ARG USER_GID=$USER_UID

RUN apt-get update && apt-get install -y --no-install-recommends \
    ros-humble-pcl-ros \
    ros-humble-tf2-eigen \
    ros-humble-rviz2 \
    # --- Added Missing C++ Dependencies ---
    libopencv-dev \
    ros-humble-vision-opencv \
    libzip-dev \
    # --------------------------------------
    build-essential \
    libeigen3-dev \
    libjsoncpp-dev \
    libspdlog-dev \
    libcurl4-openssl-dev \
    cmake \
    python3-colcon-common-extensions \
    python3-rosdep \
    && rm -rf /var/lib/apt/lists/*

RUN groupadd --gid $USER_GID $USERNAME \
    && useradd -u $USER_UID --gid $USER_GID -m $USERNAME
    
USER $USERNAME
WORKDIR /ouster_ws

COPY --chown=$USERNAME:$USERNAME ros_entrypoint.sh /ros_entrypoint.sh

RUN chmod +x /ros_entrypoint.sh

ENTRYPOINT ["/ros_entrypoint.sh"]
CMD ["bash"]
