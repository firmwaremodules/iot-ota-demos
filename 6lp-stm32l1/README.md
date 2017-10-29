# Secure OTA Firmware Update with STM32L1 over 6LoWPAN and CoAP

  ![](https://github.com/firmwaremodules/iot-ota-demos/raw/master/6lp-stm32l1/resources/ota-6lp-stm32-spirit1-cover2.jpg "NUCLEO-L152RE with X-NUCLEO-IDS01A5")

## About this project
Demonstrate a secure and robust over-the-air and end-to-end firmware update from a CoAP server over 6LoWPAN with the STM32L152RE Cortex-M3 MCU and the SPSGRF915 Spirit1 Sub-GHz RF module.
This demonstration deploys the IoT Firmware Core over-the-air firmware update technology to the STM32 family of devices running a Contiki 6LoWPAN stack.

## Project info
Difficulty: Easy  
Estimated Time: 30 minutes

## Things used in this project

### Hardware

**6LoWPAN Node**
* [STMicro NUCLEO-L152RE development board](http://www.st.com/en/evaluation-tools/nucleo-l152re.html) x1
* [STMicro X-NUCLEO-IDS01A5 sub-1 GHz RF expansion board](http://www.st.com/content/st_com/en/products/ecosystems/stm32-open-development-environment/stm32-nucleo-expansion-boards/stm32-ode-connect-hw/x-nucleo-ids01a5.html) x1

**6LoWPAN Border Router**
* [STMicro NUCLEO-F401RE development board](http://www.st.com/en/evaluation-tools/nucleo-f401re.html)
* [STMicro X-NUCLEO-IDW01M1 Wi-Fi expansion board](http://www.st.com/content/st_com/en/products/ecosystems/stm32-open-development-environment/stm32-nucleo-expansion-boards/stm32-ode-connect-hw/x-nucleo-idw01m1.html)
* [STMicro X-NUCLEO-IDS01A5 sub-1 GHz RF expansion board](http://www.st.com/content/st_com/en/products/ecosystems/stm32-open-development-environment/stm32-nucleo-expansion-boards/stm32-ode-connect-hw/x-nucleo-ids01a5.html) x1

### Software

* [OTA-STM32L1-SPIRIT1 Firmware Binary](https://github.com/firmwaremodules/iot-ota-demos/raw/master/6lp-stm32l1/OTA-STM32L1-SPIRIT1_1_0_31_MAN_0x58E59CCB.bin)
* [iot-ota-server-coap.js](https://github.com/firmwaremodules/iot-ota-server/raw/master/iot-ota-server-coap.js)

### Software Tools

* Terminal emulator such as [Tera Term](https://ttssh2.osdn.jp/index.html.en) or [PuTTY](http://www.chiark.greenend.org.uk/~sgtatham/putty/)
* [node.js](https://nodejs.org)

## Story
### Introduction
The purpose of this demonstration is to show how an AES-128 encryped firmware update image for a tiny, low-power IoT device can be retrieved from anywhere on the Internet using native IP end-to-end and installed into the internal flash of a high-availability IoT MCU device.  The device's internal flash is partitioned into 5 sections:
* A simple non-updatable bootloader in the **BOOT** section.
* A completely separate provisioning application containing features and functions intended only to be used during manufacturing and configuration of the device in the **MTA** section.
* Two equally sized executable application sections called **APP1** and **APP2**.
* A data section called **DATA** to store provisioned data and runtime data as the application requires.

There are two types of firmware image files used in this demo:
* The MAN image (short for MANufacturing) is the combination of bootloader, MTA and APP binaries and is programmed onto the device's internal flash before it is locked down and rolled off the manufacturing line.
* The APP image is the main application implementing the device's main functions.  The APP contains the OTA update logic, network stacks and/or interfaces and OS, and thus all components critical to the execution of the device are updated together.  The APP is encrypted with AES-128-CBC.  The APP can be placed into either the APP1 or APP2 sections at the discretion of the OTA update logic.

When the device is first programmed with the MAN firmware image, the application image resides in the APP1 section, and APP2 is empty.  The bootloader checks for a valid application in either section and boots the most recent (highest version) - in this case the application in section APP1.

The demo application polls a CoAP server at a configurable IPv4 address for the presence of newer firmware images.  When a new application image is available, it is downloaded through the CoAP block transfer mechanism.  The application image itself is decrypted on-the-fly with a software AES engine (the STM32L152 does not have a hardware AES engine) and stored into the APP2 section.  The update logic in this demo automatically and immediately reboots the device when the stored image has been verified.  The bootloader then determines that the new image is valid and jumps to it, completing the update process.   

If the update is interrupted at any time, for example mid-way during the download or during a flash write, or if the image could not otherwise be correctly written in its entirety to the APP2 section, the bootloader will not accept the content of APP2 and continue to boot the existing application. This is an extremely robust update process designed for IoT devices operating in real-world conditions and requires no external components.  However, it comes at a cost - the amount of internal flash available to applications is reduced by half.  Fortunately, there is plenty of flash on the STM32L15x line of MCUs (512KB) to host the full Contiki stack plus apps.

### What will happen?

* The firmware application running on the STM32L152 target MCU will be updated by downloading a new version of essentially the same application from a CoAP server that you will host and point the firmware to.

### Steps to complete the demo

0. **Setup the CoAP server**

You'll need node.js installed and the iot-ota-server-coap.js script located in a directory with the requisite dependencies (e.g. node-coap). Perhaps the simplest thing to do here is checkout the iot-ota-server respository and follow the instructions in the README.  The OTA firmware update file for this demo is already committed there, in the **updates** directory.

1. **Assemble the hardware.** Assemble the NUCLEO-L152RE and X-NUCLEO-IDS01A5 expansion board as shown in the image below and connect to your PC with a USB cable.  Connect a jumper from 3.3V to PA0 to select the provisioning application (the MTA) for initial boot so that you can set the CoAP server IPv4 address.

Think about this for a second.  There are no gateways, and no protocol translators involved here.  The data packet emitted by the device (UDP) is passed from router to router and received by any reachable address on the Internet exactly as produced.  This is the true IoT!

  ![](https://github.com/firmwaremodules/iot-ota-demos/raw/master/6lp-stm32l1/resources/ota-6lp-stm32-spirit1-hwconfig.jpg "NUCLEO-L152RE with X-NUCLEO-IDS01A5 and jumper wire")
  
You also need to setup the border router!  This is not part of the demo because in theory **any** compatible 6LoWPAN border router would work.  However, the demo firmware is built with parameters that allow it to communicate with STMicro's Wifi-Bridge border router application available as a ready-to-go binary in their [FP-NET-6LPWIFI1](http://www.st.com/content/st_com/en/products/embedded-software/mcus-embedded-software/stm32-embedded-software/stm32-ode-function-pack-sw/fp-net-6lpwifi1.html) solution.  If you download and install this package, you'll find the border router binary as Projects/Multi/Applications/WiFi-Bridge/Binary/STM32F401RE-Nucleo/STM32F4xx-Nucleo-WiFi-Bridge-IDS01A5.bin which you can load onto the NUCLEO-F401RE with the drag-n-drop method.  Attach the X-NUCLEO-IDW01M1 then the X-NUCLEO-IDS01A5 to the NUCLEO-F401RE and load the WiFi-Bridge firmware.  The firmware will prompt you to setup the WiFi connection parameters and that's all you have to do!



2. **Get the node firmware file.** Download the `OTA-STM32L1-SPIRIT1_x_x_x_MAN_x.bin` firmware binary programming file to a folder on your PC.  This binary can be programmed with the "drag-n-drop" programming method available with the embedded STLink on NUCLEO boards (mbed method).  Drag the binary to the mass storage device that pops up when you plug in the NUCLEO board.  Alternatively, you can use the firmware programming tool 'fm_load.exe', which supports programming the STM32L152RE device as well.


3. **Start the terminal.** Start a terminal emulator session/connection on the NUCLEO's "STMicroelectronics STLink Virtual COM Port (COMX)" at 115200 baud.  Choose the COM port that is assigned by your PC.

4. **Load the firmware.** Drag the MAN firmware binary `fm_load OTA-STM32L1-SPIRIT1_1_0_31_MAN_0x58E59CCB.bin` to the mass storage device opened up on your PC as a result of plugging in the NUCLEO board.  Observe loading progress to completion (100%).
 
Observe the serial terminal window's output showing the MTA provisioning application menu that we'll use to set the OTA update server IP addres with, which we'll do next.
  
5. **Provision the device's OTA update server settings.**  To do this we must boot into the device's MTA (manufacturing test application) by pulling PA0 high and resetting the board.  Connect a short jumper wire as shown between the X-NUCLEO-IDS01A5 connector pins CN6-4 (Vcc) and CN8-1 (PA0) and press the reset (blue) push-button on the NUCLEO.

  
  The serial terminal will show an increasingly option-filled menu that allows you to set Wifi, Device (Cloud) and OTA server settings.  All we need to set for this demo is the OTA server's IP address; the rest can remain unset. Enter the IPv4 address of your machine hosting the [CoAP server](https://github.com/firmwaremodules/iot-ota-server).  
  
  ```
************************************************************************
 IoT Core Provisioning MTA for NUCLEO-L152RE-SPIRIT1
************************************************************************

 WiFi Settings:

   SSID: not set
   Security Type: OPEN
   Passkey: not set

   1) Set SSID
   2) Set Security Type
   3) Set Passkey

 Device Settings:

   NOT USED: not set
   NOT USED: not set
   NOT USED: not set

   7) Set NOT USED
   8) Set NOT USED
   9) Set NOT USED

 OTA Settings:

   OTA server IP: not set

   a) Set OTA server IP address


 >
  ```

  When complete, commit the settings to flash with menu option ` 4) Save settings`.  Remove the jumper wire from the 3.3V pin but leave the other end attached - you might want to use it again later to "rollback" the update for additional testing.
  
    
6. **Perform the update.** Reset the board (ensure the jumper is disconnected) to boot back into the main application.  This time it should connect to your local 6LoWPAN network and proceed to download and install the firmware update sourced from the CoAP server as OTA-STM32L1-SPIRIT1_1_1_31_APP_0xF63D0305.fmu.  

A couple of notes to observe in the output below.
* First note that the "APP2 version" is 0.0.0, indicating that nothing is present in that section.
* The node is designed to talk to the stock "WiFi-Bridge" project binary in the STM32CubeFunctionPack_6LPWIFI_V2.0.0 function pack as Projects/Multi/Applications/WiFi-Bridge/Binary/STM32F401RE-Nucleo/STM32F4xx-Nucleo-WiFi-Bridge-IDS01A5.bin
* The update won't start until the node is connected to the border router (aka WiFi Bridge), and you can tell it is connected when you see as series of RPL info messages print in the console that indicate a preferred DAG has been joined, i.e. `RPL: preferred DAG aaaa::1151:3433:6834:6d31`.  The address will be unique to your node, however the IPv6 prefix is fixed to "aaaa", which is required to connect to the STM WiFi-Bridge border router.

  ```
************************************************************************
 IoT Core OTA Update Demo APP for NUCLEO-L152RE-SPIRIT1
************************************************************************
BOOT version: 1.0.31
MTA version: 1.0.31
APP1 version: 1.0.31
APP2 version: 0.0.0
Current APP version: 1.0.31
Client IPv6 addresses:
aaaa::3b34:3239:5136:710c
fe80::3b34:3239:5136:710c
autostart_start: starting process 'Ota Test Process'
Starting OTA Update process.
Provisioning data retrieved.
  OTA server IPv4 address: 192.168.1.64
Press the user button to initiate OTA firmware update
after connection with the border router is established.
OTA Erbium CoAP client will contact server at: ::FFFF:192.168.1.64 : 5683 /updates
autostart_start: starting process 'Ota Update Process'
RPL: New instance detected (ID=30): Joining...
RPL: rpl_add_parent lladdr @200018f4 fe80::1151:3433:6834:6d31
RPL: Adding fe80::1151:3433:6834:6d31 as a parent: succeeded
RPL: rpl_set_preferred_parent fe80::1151:3433:6834:6d31 used to be NULL
RPL: Joined DAG with instance ID 30, rank hu, DAG ID aaaa::1151:3433:6834:6d31
RPL: Adding default route through fe80::1151:3433:6834:6d31
RPL: rpl_process_parent_event recalculate_ranks
RPL: Preferred parent update, rank changed from 384 to 345
RPL: Moving in the instance from rank hu to hu
RPL: The preferred parent is fe80::1151:3433:6834:6d31 (rank 1)
RPL: Prefix announced in DIO
RPL: Prefix set - will announce this in DIOs
RPL: rpl_set_prefix - prefix NON-NULL
RPL: Set dag aaaa::1151:3433:6834:6d31 lifetime to 3145
RPL: Received consistent DIO
RPL: preferred DAG aaaa::1151:3433:6834:6d31, rank 345, min_rank 345, parent rank 128, link metric 217
RPL: rpl_process_parent_event recalculate_ranks
xdc.runtime.Main: line 210: Starting CoAP update, state=0
--Requesting /updates--
::FFFF:192.168.1.64 : 5683
  updating...392f7c38, 90608, 89504
[Updating: 256/90624 0%]
RPL: rpl_process_parent_event recalculate_ranks
RPL: Preferred parent update, rank changed from 345 to 277
[Updating: 512/90624 0%]
[Updating: 768/90624 0%]
[Updating: 1024/90624 1%]
RPL: rpl_process_parent_event recalculate_ranks
RPL: Preferred parent update, rank changed from 277 to 260
[Updating: 1280/90624 1%]
[Updating: 1536/90624 1%]
[Updating: 1792/90624 1%]
[Updating: 2048/90624 2%]
RPL: rpl_process_parent_event recalculate_ranks
RPL: Preferred parent update, rank changed from 260 to 256
[Updating: 2304/90624 2%]
[Updating: 2560/90624 2%]
[Updating: 2816/90624 3%]

....

[Updating: 88576/90624 97%]
[Updating: 88832/90624 98%]
[Updating: 89088/90624 98%]
RPL: rpl_process_parent_event recalculate_ranks
[Updating: 89344/90624 98%]
[Updating: 89600/90624 98%]
[Updating: 89856/90624 99%]
RPL: rpl_process_parent_event recalculate_ranks
[Updating: 90112/90624 99%]
[Updating: 90368/90624 99%]
COMPLETE
[Updating: 90624/90624 100%]
[Updating: 90624/90624 100%]

--Done--
Contiki going down NOW!
  ```
  This update is identical in function to the previous version, except that the version 'minor' digit has been incremented from 0 to 1, and there is an extra message in the banner indicating that the updated app is running.  When the update is complete, the board automatically reboots and you should see the updated application now running out of the STM32L152 internal flash's APP2 section.  

  ```
************************************************************************
 IoT Core OTA Update Demo APP for NUCLEO-L152RE-SPIRIT1
 Running the updated APP!
************************************************************************
BOOT version: 1.0.31
MTA version: 1.0.31
APP1 version: 1.0.31
APP2 version: 1.1.31
Current APP version: 1.1.31

  ```

7. **Rollback the update.** Re-connect the jumper wire between PA0 and 3.3V and reset the board to access the MTA.  The menu will have a new option `5) rollback a firmware update` as shown. Select this option and confirm with `y`. 

  ```
************************************************************************
 IoT Core Provisioning MTA for NUCLEO-L152RE-SPIRIT1
************************************************************************

 WiFi Settings:

   SSID: not set
   Security Type: OPEN
   Passkey: not set

   1) Set SSID
   2) Set Security Type
   3) Set Passkey

 Device Settings:

   NOT USED: not set
   NOT USED: not set
   NOT USED: not set

   7) Set NOT USED
   8) Set NOT USED
   9) Set NOT USED

 OTA Settings:

   OTA server IP: 192.168.0.1

   a) Set OTA server IP address
 
 5) Rollback a firmware update

 > 5 rollback to version 1.0.31 from 1.1.31 y/n?
  ```
10. **Interrupt the update.**  Remove the jumper wire again from the 3.3V pin and reset the board to boot the main APP.  Now this time around, you can press the reset button as the update progresses, or power cycle the board.  At this stage, a portion of the image has been written to the APP2 section in internal flash, and a flash write operation may have been ongoing.  When the application boots up again, it remains untouched and it is the same as before - the current version will continue to report 1.0.31.  


### So how does this work?

The bootloader, manufacturing test application and main application are independently developed, compiled and linked with complete vector tables.  They are assembled into the combined MAN image in a post-build ("release") step.  At the same time, the application images are embedded into a firmware update container format that allows relocation of firmware images (**.fmu**) and then encrypted.  You can confirm that the .fmu file is encrypted by noting that there is no evidence of a vector table anywhere in the file including the SP and PC initialization values typical at word offsets 0 and 1.  You could even use a hex editor to change a byte anywhere in the .fmu file, and the update will fail at the final verification stage.

The bootloader is located at the beginning of the MAN image and after programming, it is placed at the location where the Cortex-M processor expects an application to reside on startup.  The bootloader performs self-tests, checks the status of the MTA selection pin (PA0) and verifies the integrity of all application images to decide where to set the processor's VTOR (vector table offset register).  Once the VTOR is set, the PC (program counter) is moved to point to the target application's reset vector (specified at word offset 1 in the target application's vector table). The bootloader's job ends there and the application's startup procedures take over and reconfigures the device as required for the application.

If the bootloader encounters any fatal errors, it is designed to signal the error condition to port PA5 - conveniently attached to the NUCLEO-L152RE's user LED, LD2.

The main application is built with the Contiki OS.  Contiki's Erbium CoAP client, unmodified, handles the task of transfering the firmware update image with CoAP's blockwise transfer mode.  A block of 256 bytes was selected to provide a good overall data flow.  

The encryption key is stored with the application and is therefore vulnerable to inspection in the MAN image.  However, the MAN image is not normally distributed, but intended to be used only in a "trusted" product manufacturing environment (such as your workbench, lab or facility).  Once the MAN image is programmed onto the device, and the JTAG access ports have been disabled and flash protection level 2 has been enabled, the encryption key and indeed the firmware content itself is no longer easily accessible to anyone that can gain access to your firmware update file or physical access to your device.



### What's next

An end-to-end firmware update from a tiny, low-power MCU device to a server that could be hosted anywhere on Earth and speaking its native language (IP) is a huge step forward in realizing the billions-of-devices IoT that we're all looking forward to.  But this is still just the beginning.  Scaling a firmware update to support thousands (let alone billions) will require a different, more customized approach, leveraging the unique capabilities of the 6LoWPAN stack, RPL and TSCH.



