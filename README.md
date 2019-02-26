Notice: This is a work in progress, not all sections are complete.

# OpenVINO-ARM64
OpenVINO ARM64 Notes

Intel does not currently offer complete OpenVINO installations for ARM64 / AARCH64 .  
This is an opinionated writeup detailing how to run 32-bit Debian ARMHF binaries on 64-bit ARM environments using chroot.  

See:  
https://software.intel.com/en-us/forums/computer-vision/topic/801986  
https://software.intel.com/en-us/forums/computer-vision/topic/802550  
https://software.intel.com/en-us/forums/computer-vision/topic/800823  
https://github.com/opencv/dldt/issues/3  

# Version & Compatibility Notes

## OpenCV 4 and OpenVINO 2018 R5

1. OpenVINO only provides 32-bit ARM libraries for Raspbian.
2. OpenVINO binaries are only compiled for Python 2.7 and Python 3.5.
3. Python 3 is preferred over Python 2.7, many softwares are losing Python 2 compatibility.
4. OpenVINO R5 comes with OpenCV 4.0.1 compiled for Raspbian ARMHF.
5. Raspbian ARMHF modules won't always work on other Linux distributions (broken links, module signatures)
6. 

## Ubuntu, OpenCV 4, and OpenVINO 2018 R5

1. Ubuntu is recommended over Debian Stretch.  `numpy` for Python 3.X on Debian Stretch requires GLIBC 2.27+ (official repos only provide GLIBC 2.24)
2. The Intel OpenVINO libraries only work with Python 2.7 and 3.5.  Python 3.6 and 3.7 won't work.  Intel does not provide full sources at this time.
3. Bionic does not have packages for Python 3.5, add Xenial package sources for Python 3.5
4. I suggest building OpenCV from source, The `cv2.so` provided by Intel may not work in all environments.
5. Your [chroot] environment will need to load the Intel modules:
    - `sudo cp -r ./inference_engine_vpu_arm/python/python3.5/armv7l/* /usr/local/lib/python3.5/dist/paackages/`
    - `sudo mkdir -p /usr/local/lib/python3.5/dist-packages/openvino/inference_engine/`
    - `sudo cp -r ./inference_engine_vpu_arm/deployment_tools/inference_engine/lib/raspbian_9/armv7l/* /usr/local/lib/python3.5/dist-packages/openvino/inference_engine/`
 - Use `export DISPLAY=:0` to enable GUI's inside the schroot

# Installation

## 32-bit ARMHF Schroot

See:  
https://wiki.debian.org/ArmHardFloatChroot  
https://help.ubuntu.com/community/DebootstrapChroot

**Ubuntu Bionic Schroot** (Recommended)  
```
# Install the base environment, run:
sudo debootstrap --include=build-essential,sudo,apt,nano --arch=armhf bionic /srv/chroot/bionic_armhf/ http://ports.ubuntu.com/ubuntu-ports/

# Enter the environment, run:
schroot -e bionic_armhf

# Verify the following are in /etc/apt/sources.list
deb http://ports.ubuntu.com/ubuntu-ports/ bionic main restricted universe multiverse
deb-src http://ports.ubuntu.com/ubuntu-ports/ bionic main restricted universe multiverse
deb http://ports.ubuntu.com/ubuntu-ports/ bionic-security main restricted universe multiverse
deb-src http://ports.ubuntu.com/ubuntu-ports/ bionic-security main restricted universe multiverse
deb http://ports.ubuntu.com/ubuntu-ports/ bionic-updates main restricted universe multiverse
deb-src http://ports.ubuntu.com/ubuntu-ports/ bionic-updates main restricted universe multiverse
deb http://ports.ubuntu.com/ubuntu-ports/ xenial main restricted universe multiverse
```

**Notes**  
1. Use `export DISPLAY=:0` to enable display and GUI inside the schroot.
2. `xenial` source is added to Bionic for fetching `python3.5`
3. Run `sudo mount --rbind /dev /srv/chroot/bionic_armhr/dev` to allow USB access inside the schroot.


## Compiling 

## Contributing

Feel free to fork and create a pull request or an issue if you have additional materials to add.
