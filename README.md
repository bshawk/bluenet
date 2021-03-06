# Indoor Localization with BLE

This project aims at a wireless network with BLE nodes that use their mutual signal strengths to build up a network with their relative locations. This can be used later by someone carrying a smartphone to establish their location indoors. Everybody say that they can do it, but very few solutions are actually out there. Let's hope we can change that.

Bluetooth LE (BLE) does not inherently fit a wireless network. We have technology in-house (at the Almende group) that can do this [https://en.wikipedia.org/wiki/MyriaNed](Myrianed), but it has not been accepted in the mainstream yet. By the way, it is my personal opinion that solutions such as ZigBee, Z-Wave, MyriaNed, and other mesh solutions, will remain marginal except if they get accepted in a common handheld.

That's why BLE is interesting. A lot of phones come with BLE, so a solution is automatically useful to a large variety of people. It is not the best technology for the job. The network topology is a Personal Area Network (PAN), not a Local Area Network (LAN). This means that you cannot have all nodes communicating with all other nodes at the same time. To get RSSI values we will have to set up connections to other nodes and tear them down again. Not very efficient. But it will do the job. The new SoftDevice from Nordic (the S120 instead of the S110) might change this, but it is still in its alpha stage. So, for now we consider a network with known nodes, setting up connections amongst them will be relatively easy. On the moment I am not concerned with security so establishing some zeroconf method to detect which nodes belong to the network will not be part of this codebase yet. My idea however is to do this through synchronizing over the main switch in a home. Turn everything off and the network starts searching for a smartphone who is advertising authentication to use. When this is set, it will be used next time the power goes down. 

Feel free to clone this repos.

The code base comes from [http://hg.cmason.com/nrf](http://hg.cmason.com/nrf). Thanks a lot Christopher!

## Installation

The installation should not be hard when you have the Nordic SDK. Get this from their website after buying a development kit. You also need a cross-compiler for ARM. You need the JLink utilities from Segger. And you need cmake for the build process.

* [Nordic nRF51822 SDK](https://www.nordicsemi.com/eng/Products/Bluetooth-R-low-energy/nRF51822)
* [Nordic S110 Softdevice](http://www.nordicsemi.com/eng/Products/S110-SoftDevice-v7.0)
* [JLink Software](http://www.segger.com/jlink-software.html)
* sudo aptitude install cmake

A cross-compiler for ARM is the `GCC` cross-compiler which is maintained by the ARM folks on [Launchpad](https://launchpad.net/gcc-arm-embedded/4.8/4.8-2014-q3-update/+download/gcc-arm-none-eabi-4_8-2014q3-20140805-src.tar.bz2).

### Bugs

There is a bug in one of the SDK files, namely `nrf_svc.h` (different location depending on the SDK version):

    /opt/nrf51_sdk/v6/nrf51822/Include/s110/nrf_svc.h
    /opt/nrf51_sdk/v4/nrf51822/Include/ble/softdevice/nrf_svc.h

Change the assembly line:

        "bx r14" : : "I" (number) : "r0" \

Into:

        "bx r14" : : "I" ((uint16_t)number) : "r0" \

## Usage

You will have to attach a programmer/debugger somehow. Towards that you only need four pins. On the RFduino this is `GND`, `3V`, `RESET`, and `FACTORY` and they are subsequent pins on that side of the RFduino where there are most pins (the other side has the antenna stealing a bit of space for eventual pins). The pin layout of the JLink connector is written out on the [DoBots blog](http://dobots.nl/2014/03/05/rfduino-without-rfduino-code/).

On the moment the current functionality is extremely simple. The service `00002220-0000-1000-8000-00805f9b34fb` provides a write characteristic `0124` to which can be written a value. If you attach a devices to pin `GPIO5` you will see it toggled.

### Configuration

Fork the code by clicking on:

* Fork [https://github.com/mrquincle/bluenet/fork](https://github.com/mrquincle/bluenet/fork).
* `git clone https://github.com/${YOUR_GITHUB_USERNAME}/bluenet`
* let us call this directory $BLUENET

Now you will have to set all fields in the configuration file:

* cp CMakeBuild.config.default CMakeBuild.config
* adjust the `NRF51822_DIR` to wherever you installed the Nordic SDK (it should have `/Include` and `/Source` subdirectories
* adjust the `SOFTDEVICE_DIR` to wherever you unzipped the latest SoftDevice from Nordic
* adjust the `SOFTDEVICE_SERIES` to for example `110` or `130` (SoftDevice name starts with it, without the `s`)
* adjust the `SOFTDEVICE_DIR_API` to the directory with the SoftDevice include files
* set `SOFTDEVICE_NO_SEPARATE_UICR_SECTION=1` if you use one of the earlier s110 softdevices with a separate UICR section
* adjust the type `SOFTDEVICE` accordingly (basename of file without `_softdevice.hex`)
* set the `APPLICATION_START_ADDRESS` to start of application in FLASH (called `CODE_R1_BASE` in Nordic documentation)
* set the `APPLICATION_LENGTH` to what remains of FLASH 
* set `RAM_R1_BASE` to the start of RAM that is available (SoftDevice S130 v0.5 uses a staggering 10kB from the 16kB!)
* set `RAM_APPLICATION_AMOUNT` to what remains for the application in RAM
* adjust the `COMPILER_PATH` and `COMPILER_TYPE` to your compiler (it will be used as `$COMPILER_PATH\bin\$COMPILER_TYPE-gcc`)
* adjust `JLINK` to the full name of the JLink utility (JLinkExe on Linux)
* adjust `JLINK_GDB_SERVER` to the full name of the JLink utility that supports gdb (JLinkGDBServer on Linux)

Let us now install the SoftDevice on the nRF51822:

* cd scripts
* ./softdevice.sh build
* ./softdevice.sh upload

Now we can build our own software:

* cd $BLUENET
* make

And we can upload it:

* cd scripts
* this would do the same as building above ./firmware build crownstone
* ./firmware.sh upload crownstone
* ./firmware.sh debug crownstone

You can also run:

* ./firmware.sh all crownstone

And there you go. There are some more utility scripts, such as `reboot.sh`. Use as you wish. 

## UART

Currently UART for debugging. In case you happen to have the nRFgo Motherboard (nRF6310, strongly recommended) you can 
easily connect the pints at P2.0 and P2.1 to respectively the pins RXD and TXD on P15 on the board. Do not forget to
switch on the RS232 switch. Subsequently you will need some RS232 to USB cable if you haven't an extremely old laptop.
The current set baudrate you can find in `src/serial.cpp` and is `38400` baud. To read from serial, my personal 
favorite application is  `minicom`, but feel free to use any other program.

    (sudo) minicom -c on -s -D /dev/ttyUSB0

The sudo rights are only necessary if your `udev` rights are not properly set. The above flags just set colors to `on`
and define the right usb port to use (if you've multiple).

## Todo list

* Clean up code
* Obtain RSSI values
* Set up management for establishing connections, getting RSSI values, and tear down connections again
* Create an algorithm to come up with all locations of the nodes. These positions are relative, not absolute. The network will be known up to rotation and scale.
* Implement this algorithm in a distributed fashion. The type of algorithm I have in mind is belief propagation / message passing.

## Commercial use

This code is used in a commercial product at DoBots, the [Crownstone](http://dobots.nl/products/crownstone). Our intellectual property exists on two levels. First, you can license our technology to create these extremely cheap BLE building blocks yourself. Second, we build services around BLE-enabled devices. This ranges from smartphones to [gadgets such as the "virtual memo"](http://dobots.nl/2014/07/15/ble-dobeacon-a-virtual-memo/). What this means for you as a developer is that we can be transparent about the software on the Crownstone, which is why this repository exists. Feel free to build your own services on top of it, and benefit from our software development as much as you want. 

It would be much appreciated to state "DoBots inside" in which case we will be happy to provide support to your organisation.

## Copyrights

Obviously, the copyrights of the code written by Christopher, belong to him.

The copyrights (2014) for the rest of the code belongs to the team of Distributed Organisms B.V. and are provided under an noncontagious open-source license:

* Authors: Anne van Rossum, Dominik Egger, Bart van Vliet
* Date: 27 Jan. 2014
* License: LGPL v3+, Apache, or MIT, your choice
* Almende B.V., http://www.almende.com and DoBots B.V., http://www.dobots.nl
* Rotterdam, The Netherlands

Note, that we do not use any header files from Nordic. The header files are rewritten from scratch by Christopher 
especially for that purpose. This means you are not finding a Nordic license text in this repository. 
Of course, this means that you will have to get those files from Nordic via different means (we recommend to buy the
development kit). We just didn't want to "contaminate" this repository with files that we don't understand the license
implications of.
