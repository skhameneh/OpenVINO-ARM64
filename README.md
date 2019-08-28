# OpenVINO-ARM64
OpenVINO ARM64 Notes

Intel currently does not offer OpenVINO for ARM64 / AARCH64.  
This is an opinionated writeup detailing how to run 32-bit Debian ARMHF binaries on a 64-bit ARM Ubuntu host using schroot.

See:  
https://software.intel.com/en-us/forums/computer-vision/topic/801986  
https://software.intel.com/en-us/forums/computer-vision/topic/802550  
https://software.intel.com/en-us/forums/computer-vision/topic/800823  
https://github.com/opencv/dldt/issues/3  

# Version & Compatibility Notes

## Ubuntu, OpenCV 4.1.1 and OpenVINO 2019 R2

1. OpenVINO only provides 32-bit 'armhf' libraries for Raspbian ('armhf' = Arm port using the hard-float ABI).
2. OpenVINO binaries are only compiled for Python 2.7 and Python 3.5/3.7.
3. Python 3 is preferred over Python 2.7 because of [this](https://pythonclock.org).
4. OpenVINO R5 comes with OpenCV 4.1.1 compiled for Raspbian.
5. Raspbian armhf modules are not fully compatible with other Linux distributions (broken links, dependencies, naming/version hell).
6. Intel does not provide all sources for the OpenVINO libraries at this time.
7. I suggest building OpenCV from source. The `cv2.so` provided by Intel may not work in all environments.

# Installation

## 1. Install prerequisites

```
$ sudo apt install debootstrap schroot
```

See:
https://wiki.debian.org/ArmHardFloatChroot  
https://help.ubuntu.com/community/DebootstrapChroot

## 2. Create 32-bit armhf Debian 10 (Buster) schroot

```
$ sudo debootstrap --include=build-essential,sudo,apt,nano --arch=armhf buster /srv/chroot/buster_armhf/ http://ftp.debian.org/debian
```

Open the file `/etc/schroot/schroot.conf` and add the following lines:

```
[buster_armhf]
description=buster_armhf
type=directory
directory=/srv/chroot/buster_armhf
users=MYUSER
```

Replace "MYUSER" with your username.

## 3. Download OpenVINO 2019 R2 and configure host/schroot environment

Download OpenVINO 2019 R2 and copy the content of the archive into the schroot environment:

```
$ cd ~/Downloads
$ wget https://download.01.org/opencv/2019/openvinotoolkit/R2/l_openvino_toolkit_runtime_raspbian_p_2019.2.242.tgz
$ tar -xf l_openvino_toolkit_runtime_raspbian_p_2019.2.242.tgz
$ sudo cp -R ./l_openvino_toolkit_runtime_raspbian_p_2019.2.242 /srv/chroot/buster_armhf/
```

Add the user to the `users` group:

```
$ sudo usermod -a -G users "$(whoami)"
```

Add udev rules for the NCSx stick:

```
$ source ./l_openvino_toolkit_runtime_raspbian_p_2019.2.242/bin/setupvars.sh
$ sh ./l_openvino_toolkit_runtime_raspbian_p_2019.2.242/install_dependencies/install_NCS_udev_rules.sh
```

Copy the OpenVINO binaries to the schroot environment:

```
$ sudo mkdir -p /srv/chroot/buster_armhf/usr/local/lib/python3.7/dist-packages
$ sudo cp -R ./l_openvino_toolkit_runtime_raspbian_p_2019.2.242/python/python3.7/* /srv/chroot/buster_armhf/usr/local/lib/python3.7/dist-packages/
$ sudo cp -R ./l_openvino_toolkit_runtime_raspbian_p_2019.2.242/deployment_tools/inference_engine/lib/armv7l/* /srv/chroot/buster_armhf/usr/local/lib/python3.7/dist-packages/openvino/inference_engine/
```

Allow USB access inside the schroot environment:

```
$ sudo mount --rbind /dev /srv/chroot/buster_armhf/dev
```

Enter the schroot environment:

```
$ schroot -c buster_armhf
```

Your prompt should now show "(buster_armhf)" in front of your username. This indicates that you are now inside the schroot environment.

**WARNING:** Make sure that you are really inside the schroot environment. Otherwise you will mess with the configuration of your host system!

Note: Your user's home directory is automatically mapped to the schroot environment.

Add the user to the `users` group:

```
$ sudo usermod -a -G users "$(whoami)"
```

## 4. Compile OpenCV 4.1.1 from source

### Prerequisites

```
$ sudo apt update
$ sudo apt install wget python3-dev libgtk-3-dev libatlas-base-dev gfortran \
    build-essential cmake unzip pkg-config \
    libjpeg-dev libpng-dev libtiff-dev \
    libavcodec-dev libavformat-dev libswscale-dev libv4l-dev \
    libxvidcore-dev libx264-dev git pkg-config \
    python3-pip libtbb2 libtbb-dev libdc1394-22-dev
$ sudo apt clean
```

Install Numpy:

```
$ sudo python3 -m pip install numpy
```

### Download and prepare OpenCV 4.1.1

```
$ cd ~/Downloads
$ wget -O opencv.zip https://github.com/opencv/opencv/archive/4.1.1.zip
$ wget -O opencv_contrib.zip https://github.com/opencv/opencv_contrib/archive/4.1.1.zip
$ unzip opencv.zip
$ unzip opencv_contrib.zip
$ mv opencv-4.1.1/ opencv/
$ mv opencv_contrib-4.1.1/ opencv_contrib/
```

### Compile and install OpenCV 4.1.1

```
$ mkdir -p opencv/build
$ cd opencv/build
$ cmake -D CMAKE_INSTALL_PREFIX=/usr/local \
      -D PYTHON3_EXECUTABLE=/usr/bin/python3 \
      -D PYTHON3_LIBRARY=/usr/lib/python3.7/config-3.7m-arm-linux-gnueabihf/libpython3.7m.so \
      -D PYTHON3_INCLUDE_DIR=/usr/include/python3.7m \
      -D PYTHON3_PACKAGES_PATH=/usr/lib/python3/dist-packages \
      -D PYTHON_DEFAULT_EXECUTABLE=/usr/bin/python3 \
      -D BUILD_OPENCV_PYTHON3=yes \
      -D OPENCV_EXTRA_MODULES_PATH=~/Downloads/opencv_contrib/modules \
      -D CMAKE_BUILD_TYPE=Release \
      -D WITH_IPP=OFF \
      -D BUILD_TESTS=OFF \
      -D BUILD_PERF_TESTS=OFF \
      -D BUILD_EXAMPLES=OFF \
      -D ENABLE_PRECOMPILED_HEADERS=OFF \
      -D ENABLE_NEON=ON \
      -D WITH_INF_ENGINE=ON \
      -D INF_ENGINE_LIB_DIRS="/l_openvino_toolkit_runtime_raspbian_p_2019.2.242/deployment_tools/inference_engine/lib/armv7l" \
      -D INF_ENGINE_INCLUDE_DIRS="/l_openvino_toolkit_runtime_raspbian_p_2019.2.242/deployment_tools/inference_engine/include" \
      -D CMAKE_FIND_ROOT_PATH="/l_openvino_toolkit_runtime_raspbian_p_2019.2.242/" \
      -D ENABLE_CXX11=ON ..
$ make -j4
$ sudo make install
```

Note: If this fails due to some strange error it is most certainly caused by the make process running out of memory. Consider creating a swap file to extend the available memory.

## 5. Test

Test the face detection demo from the official documentation:

https://docs.openvinotoolkit.org/latest/_docs_install_guides_installing_openvino_raspbian.html#run-inference-opencv

If the face detection demo is not able to find the attached NCSx stick you have to exit from the schroot and log out from the host system and login in again.

## Notes

1. Use 

```
$ export DISPLAY=:0
```

to enable display and GUI inside the schroot environment.

2. If you see a message like

```
error: (-215:Assertion failed) in function 'initPlugin'
> Failed to initialize Inference Engine backend: Cannot find plugin to use :Tried load plugin : myriadPlugin,  error: Plugin myriadPlugin cannot be loaded: cannot load plugin: myriadPlugin from : Cannot load library 'libmyriadPlugin.so': libmyriadPlugin.so: cannot open shared object file: No such file or directory, skipping
```

while trying to execute a Python script inside the schroot environment, you need to set the environment variable LD_LIBRARY_PATH. Run the following command from inside the schroot environment:

```
$ export LD_LIBRARY_PATH="/usr/local/lib/python3.7/dist-packages/openvino/inference_engine"
```

## Contributing

Feel free to fork and create a pull request or an issue if you have additional materials to add.
