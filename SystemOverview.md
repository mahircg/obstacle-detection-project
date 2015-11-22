# Introduction #
This document explains the architecture of the system, as well as the tools that we use to achieve our goal.

External modules that we use in system and information regarding to our test hardware can also be found here.

For our customized design, we use FPGA IC to design the system. On top of it, our application will run to detect the obstacles and output the warning signals, which may be considered as one of the few sequential parts of the system. Main motivation here is; unlike using a ARM-based board,which becomes quite common day by day, we would like to apply undistortioning, rectification, alignment and  correspondence parts concurrently instead of a sequential manner.


# Architecture of the System #

In first revision of the system, aim is to capture the frames via camera interface and store them in RAM. After successively completing the first part, same task will be accomplished with two camera devices. Afterwards, aim is to design and implement the logic for generating the depth map. Final version of the system will be able to generate audio warning messages using depth information in the system.

Instead of using a soft-core such as Microblaze for controller part of the system, we decided to write our own control logic by designing a FSM.

Overview of the system can be found in Figure 1.

# Hardware #

For development, we use Atlys' Xilinx Spartan-6 development board. It has a 324-pin Spartan-6 FPGA, with a lot of IO ports in it, a VHDCI port in particular which comes handy in image acqusition. It is perfectly suitable for our application, since it has AC-97 codec with audion IO pins.

For image acqusition, we use two Omnivision OV7725 CMOS sensors. Combined with an AL422 FIFO buffer, this camera module provides VGA frames up to 30 fps. We will use 15 fps in our application though. However, this module or FIFO core does not have any VHDL driver implementation or camera interface around web, so we implemented the camera interface and it will be our first open source core in this repository.

Lastly, for testing, we will use Digilent's VMod Breadboard, in which we can connect our 20-pin camera module to breadboard and transfer the frames via VMod's VHDCI interface.

As description language, VHDL shall be used. Along with Xilinx's free ISE WebPack, CoreGen will be used for generating necessary cores related with data storage, clock management, visual/audio IO and bus system. Analyzing the system and sub-modules is still an open question, since ChipScope comes with only a 30 day evaluation license. A physical logic analyzer might be considered as the core part of debugging the system.

