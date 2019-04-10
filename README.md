Notice: This is a **work in progress**, not all sections are complete.

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

## OpenCV 4.0.1 and OpenVINO 2018 R5

1. OpenVINO only provides 32-bit ARM libraries for Raspbian.
2. OpenVINO binaries are only compiled for Python 2.7 and Python 3.5.
3. Python 3 is preferred over Python 2.7 because of [this](https://pythonclock.org).
4. OpenVINO R5 comes with OpenCV 4.0.1 compiled for Raspbian ARMHF.
5. Raspbian ARMHF modules are not fully compatible with other Linux distributions (broken links, dependencies, naming/version hell)
6. ...

## Ubuntu, OpenCV 4, and OpenVINO 2018 R5

1. Ubuntu is recommended over Debian Stretch. `numpy` for Python 3.x on Debian Stretch requires GLIBC 2.27+ (official repos only provide GLIBC 2.24)
2. The Intel OpenVINO libraries only work with Python 2.7 and 3.5. Python 3.6 and 3.7 are not supported. Intel does not provide all sources at this time.
3. Bionic does not have packages for Python 3.5, so Xenial package sources are needed to install Python 3.5.
4. I suggest building OpenCV from source. The `cv2.so` provided by Intel may not work in all environments.

# Installation

## 1. Install prerequisites

```
$ sudo apt install debootstrap schroot
```

See:
https://wiki.debian.org/ArmHardFloatChroot  
https://help.ubuntu.com/community/DebootstrapChroot

## 2. Create 32-bit ARMHF Ubuntu Bionic Schroot

```
$ sudo debootstrap --include=build-essential,sudo,apt,nano --arch=armhf bionic /srv/chroot/bionic_armhf/ http://ports.ubuntu.com/ubuntu-ports/
```

Open the file `/etc/schroot/schroot.conf` and add the following lines:

```
[bionic_armhf]
description=bionic_armhf
type=directory
directory=/srv/chroot/bionic_armhf
users=MYUSER
```

Replace "MYUSER" with your username.

## 3. Download OpenVINO 2018 R5 and configure host/schroot environment

Download OpenVINO 2018 R5 and copy the content of the archive into the schroot environment:

```
$ cd ~/Downloads
$ wget http://download.01.org/openvinotoolkit/2018_R5/packages/l_openvino_toolkit_ie_p_2018.5.445.tgz
$ tar -xf l_openvino_toolkit_ie_p_2018.5.445.tgz
$ sudo cp -R ./inference_engine_vpu_arm /srv/chroot/bionic_armhf/
```

Add the user to the `users` group:

```
$ sudo usermod -a -G users "$(whoami)"
```

Add udev rules for the NCSx stick:

```
$ sh ./inference_engine_vpu_arm/install_dependencies/install_NCS_udev_rules.sh
```

Copy the OpenVINO binaries to the schroot environment:

```
$ sudo cp -R ./inference_engine_vpu_arm/python/python3.5/armv7l/* /usr/local/lib/python3.5/dist-packages/`
$ sudo cp -R ./inference_engine_vpu_arm/deployment_tools/inference_engine/lib/raspbian_9/armv7l/* /usr/local/lib/python3.5/dist-packages/openvino/inference_engine/
```

Allow USB access inside the schroot environment:

```
$ sudo mount --rbind /dev /srv/chroot/bionic_armhf/dev
```

Enter the schroot environment:

```
$ schroot -c bionic_armhf
```

Your promt should now show "(bionic_armhf)" in front of your username. This indicates that you are now inside the schroot environment.
**WARNING:** Make sure that you are really inside the schroot environment. Otherwise you will mess with the configuration of your host system!
Note: Your user's home directory is automatically mapped to the schroot environment.

Add the user to the `users` group:

```
$ sudo usermod -a -G users "$(whoami)"
```

Open the file `/etc/apt/sources.list` and replace its content with the following:

```
deb http://ports.ubuntu.com/ubuntu-ports/ bionic main restricted universe multiverse
deb-src http://ports.ubuntu.com/ubuntu-ports/ bionic main restricted universe multiverse
deb http://ports.ubuntu.com/ubuntu-ports/ bionic-security main restricted universe multiverse
deb-src http://ports.ubuntu.com/ubuntu-ports/ bionic-security main restricted universe multiverse
deb http://ports.ubuntu.com/ubuntu-ports/ bionic-updates main restricted universe multiverse
deb-src http://ports.ubuntu.com/ubuntu-ports/ bionic-updates main restricted universe multiverse
deb http://ports.ubuntu.com/ubuntu-ports/ xenial main restricted universe multiverse
```

## 4. Compile OpenCV 4.0.1 from source

### Prerequisites

```
$ sudo apt install -y \
    wget python3.5-dev libgtk-3-dev libatlas-base-dev gfortran \
    build-essential cmake unzip pkg-config \
    libjpeg-dev libpng-dev libtiff-dev \
    libavcodec-dev libavformat-dev libswscale-dev libv4l-dev \
    libxvidcore-dev libx264-dev git pkg-config \
    python3-pip libtbb2 libtbb-dev libjasper-dev libdc1394-22-dev
```

Create a link to Python 3.5:

```
$ sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.5 100
```

Install Numpy:

```
$ sudo python3 -m pip install numpy
```

### Download and prepare OpenCV 4.0.1

```
$ cd ~/Downloads
$ wget -o opencv.zip https://github.com/opencv/opencv/archive/4.0.1.zip
$ wget -o opencv_contrib.zip https://github.com/opencv/opencv_contrib/archive/4.0.1.zip
$ unzip opencv.zip
$ unzip opencv_contrib.zip
$ mv opencv-4.0.1/ opencv/
$ mv opencv_contrib-4.0.1/ opencv_contrib/
```

### Compile and install OpenCV 4.0.1

```
$ mkdir -p opencv/build
$ cd opencv/build
$ cmake -D CMAKE_INSTALL_PREFIX=/usr/local \
      -D PYTHON3_EXECUTABLE=/usr/bin/python3.5 \
      -D PYTHON3_LIBRARY=/usr/lib/python3.5/config-3.5m-arm-linux-gnueabihf/libpython3.5m.so \
      -D PYTHON3_INCLUDE_DIR=/usr/include/python3.5m \
      -D PYTHON3_PACKAGES_PATH=/usr/lib/python3/dist-packages \
      -D CMAKE_INSTALL_PREFIX=/usr/local \
      -D WITH_INF_ENGINE=ON \
      -D ENABLE_CXX11=ON \
      -D PYTHON_DEFAULT_EXECUTABLE=/usr/bin/python3.5 \
      -D BUILD_OPENCV_PYTHON3=yes \
      -D OPENCV_EXTRA_MODULES_PATH=~/Downloads/opencv_contrib/modules
      -DCMAKE_BUILD_TYPE=Release \
      -DWITH_IPP=OFF \
      -DBUILD_TESTS=OFF \
      -DBUILD_PERF_TESTS=OFF \
      -DENABLE_NEON=OFF \
      -DWITH_INF_ENGINE=ON \
      -DINF_ENGINE_LIB_DIRS="/inference_engine_vpu_arm/deployment_tools/inference_engine/lib/raspbian_9/armv7l" \
      -DINF_ENGINE_INCLUDE_DIRS="/inference_engine_vpu_arm/deployment_tools/inference_engine/include" \
      -DCMAKE_FIND_ROOT_PATH="/inference_engine_vpu_arm/" \
      -DENABLE_CXX11=ON ..
$ make -j4
$ sudo make install
```

## 5. Test

Test the face detection demo from the official documentation:

https://software.intel.com/en-us/articles/OpenVINO-Install-RaspberryPI#face-detection-model

If the face detection demo is not able to find the attached NCSx stick you have to exit from the schroot and log out from the host system and login in again.

## Notes

Use 

```
$ export DISPLAY=:0
```

to enable display and GUI inside the schroot.

## Contributing

Feel free to fork and create a pull request or an issue if you have additional materials to add.
