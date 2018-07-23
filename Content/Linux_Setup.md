# Intel(R) Media SDK 2018 Q2.1 Installation (Ubuntu 16.04)
There are several components which need to be installed in order to use the Media SDK on Linux:
 - [libVA API](https://github.com/intel/libva)
 - [Intel(R) Graphics Memory Management Library](https://github.com/intel/gmmlib)
 - [Intel(R) Media Driver for VAAPI](https://github.com/intel/media-driver)
 - [Intel® Media SDK](https://github.com/Intel-Media-SDK/MediaSDK)

## Install Ubuntu & Dependencies
The following instructions assume a fresh and fully updated installation of Ubuntu 16.04 using the HWE rolling kernel. For the Media SDK to work on newer platforms kernel **4.14.x or higher** is required. At the time of writing the latest HWE kernel is 4.15.x. You can check the kernel version using the below command:
``` bash
uname -r
```
Run the command below to install the required dependencies:
``` bash
sudo apt-get -y install git libssl-dev dh-autoreconf cmake libgl1-mesa-dev libpciaccess-dev
```
Create a working directory and export the path:
``` bash
mkdir <path-to-working-directory>
export WORKDIR=<path-to-working-directory>
```

## libVA
Run the following commands to build and install the libVA library:
``` bash
cd $WORKDIR
git clone https://github.com/intel/libva.git
cd libva
git checkout d6fd111e2062bb4732db8a05ed55fc01771087b4
./autogen.sh --prefix=/usr --libdir=/usr/lib/x86_64-linux-gnu
make -j4
sudo make install
```
## libVA Utils
Run the following commands to build and install libVA utils:
``` bash
cd $WORKDIR
git clone https://github.com/intel/libva-utils.git
cd libva-utils
git checkout 8a6ef9ed905c0d9d5463c17c76609dba5dfb9c15
./autogen.sh --prefix=/usr --libdir=/usr/lib/x86_64-linux-gnu
make -j4
sudo make install
```
