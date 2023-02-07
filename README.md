# Rowhammer Tester

# My notes for building a working example using a ZCU-104 board.

This is a working setup for my experiments

```
OS: NAME="Ubuntu"
VERSION="20.04.3 LTS (Focal Fossa)"

Xilinx tools: 
Vivado v2021.2 (64-bit)
SW Build 3367213 on Tue Oct 19 02:47:39 MDT 2021
IP Build 3369179 on Thu Oct 21 08:25:16 MDT 2021

Board: zcu104
```

## Installation
1. Installation:
```
apt install git build-essential autoconf cmake flex bison libftdi-dev libjson-c-dev libevent-dev libtinfo-dev uml-utilities python3 python3-venv python3-wheel protobuf-compiler libcairo2 libftdi1-2 libftdi1-dev libhidapi-hidraw0 libhidapi-dev libudev-dev pkg-config tree zlib1g-dev zip unzip

apt  install gcc-aarch64-linux-gnu // mandatory for building litex server properly
```

2. Clone and make dependencies:
```
git clone --recursive https://github.com/tanvirarafin/rowhammer-tester.git
cd rowhammer-tester
make deps
```

3. Set up the paths
```
source venv/bin/activate
source /PATH/TO/Vivado/VERSION/settings64.sh
```

4. Set up the variables
```
export TARGET=zcu104
export IP_ADDRESS=192.168.100.50  
```

5. Now build the bit stream
```
make build
make  test
```
The resulting bitstream file will be located in `build/zcu104/gateware/xilinx_zcu104.bit`. Rename it to zcu104.bit. 

6. Now make the EtherBone 

Make sure you are using gcc-aarch64-linux-gnu for building

```
cd firmware/zcu104/etherbone/
make LDFLAGS=-static
```

Or you can use the `zcu104_etherbone` binary already present in the `firmware/zcu104/etherbone/build` folder

7. Create the SD card:
Start with the prebuilt image in the `prebuilt/` folder. Use the following to create the SD card. 

```
sudo dd status=progress oflag=sync bs=4M if=zcu104.img of=<DEVICE>
```

We will need to update the `zcu104.bit` and the  `zcu104_etherbone` later to make the image work as intended.

8. Network setup
Put in the SD card in ZCU104. To make the ZCU104 boot from the SD card it is necessary to ensure proper switch configuration. The mode switch (SW6) consisting of 4 switches is located near the FMC LPC Connector (J5) (the same side of the board as USB, HDMI, Ethernet). For details, see “ZCU104 Evaluation Board User Guide (UG1267)”. To use an SD card, configure the switches as follows:
	1. ON
	2. OFF
	3. OFF
	4. OFF

Connect the board with an ethernet wire. 

Now change your Ubuntu network setting manually (Go to Wired setting > IPV4) Select IPV4 as manual and set your address to `192.168.100.100` and netmask to `255.255.255.0`. Hit apply. Restart the network by hitting the toggle button on the wired setting.

Turn on the SD card. It should start with a blinking pattern.

First check if the network connection is okay by pinging to  the board. In the host machine type
```
ping 192.168.100.50
```

If everything went well so far you should be able to see ping response. If not check your network connection to make sure your IP address is set to `192.168.100.100`

Now connect the micro-usb cable  with the board. Use picocom for communication

```
picocom -b 115200 /dev/ttyUSB1
```  

9. Change the bit file and EtherBone files:

a. Ssh to the board
```
ssh root@192.168.100.50
```
b. Use scp to load the new bit-stream:
```
scp build/zcu104/gateware/zcu104.bit root@192.168.100.50:/boot/zcu104.bit
```
c. Kill EtherBone on the board:
```
killall zcu104_etherbone
```
d. Now use scp to replace the EtherBone file
```
scp firmware/zcu104/etherbone/build/zcu104_etherbone  root@192.168.100.50:/bin/zcu104_etherbone 
```

e. Reboot the board. If everything goes well you should see EtherBone OK in the boot process via the serial connection just before log in prompt.

## Running the experiments:
If your EtherBone server is running properly then proceed with the experiments.

First try the leds.py example in the row hammer-tester/scipts folder.

For this you will need to run three terminals
```
Terminal 1. Connect the serial prompt to see the experiments
picocom -b 115200 /dev/ttyUSB1
```
Terminal 2. Inside the rowhammer-tester folder type the following
```
source venv/bin/activate
export TARGET=zcu104
export IP_ADDRESS=192.168.100.50  

make srv
```
If everything works then you will see the following in terminal 2. There would not be any error on the output and it will wait as an open server.
```
litex_server --udp --udp-ip 192.168.100.50 --udp-port 1234
[CommUDP] ip: 192.168.100.50 / port: 1234 / tcp port: 1234
```
Terminal 3.
In this terminal you will run the experiment. Type the following:
```
source venv/bin/activate
export TARGET=zcu104
export IP_ADDRESS=192.168.100.50  
cd rowhammer-tester/rowhammer_tester/scripts/
python leds.py 
```
This will give the following output and the light pattern on the leds will change. 
```
Using generated target files in: ../../build/zcu104
Board info: Row Hammer Tester SoC on xczu7ev-ffvc1156-2-i, git: 88f83152ee7306fc5288f87ad4a63efa2f772dc1 2023-02-06 16:25:38
```
You will also see something like `Connected with 127.0.0.1:41292` in terminal 2

Further experiments
In terminal 3 you can ctrl+C the leds.py experiment and run others. Such as
```
python hw_rowhammer.py --all-rows --nrows 5
```
You should not see any hangups rather a fast filling of the memory and then error counts etc.


### My copy of a working installation (lab.arafin) is here: https://drive.google.com/file/d/1cS-Yh-WFwy9g8STO96wnEi2isYhNF281/view?usp=sharing

# Copyright
Copyright (c) 2020-2022 [Antmicro](https://www.antmicro.com)

The aim of this project is to provide a platform for testing the [DRAM "Row Hammer" vulnerability](https://users.ece.cmu.edu/~yoonguk/papers/kim-isca14.pdf).

The repository includes:

* `rowhammer_tester/` - Core part of the project, a Python module including:

  * gateware for Rowhammer Tester platform
  * userspace scripts used for running tests
* `doc/` - Sphinx-based documentation for the project
* `.github/` - Directory with CI configuration

Full documentation is available [on Read The Docs](https://rowhammer-tester.readthedocs.io/en/latest/).
