Aim of this project is to design and implement the embedded system, which will be able to detect the obstacles from the visible range cameras. System consists of two parts; sensors and processors. Sensor part is where the two CMOS camera modules are attached to the glass, that user is wearing, and processor part is the PCB, which carries our customized design, and running the obstacle detection algorithm.

Basic methodology of the application is as follows; by using two physically aligned camera sensors, system shall generate disparity map and warn the user accordingly, with discrete type of audio warning messages such as obstacle on left,right,etc. What makes this project an academic one is not the FPGA implementation of a stereo vision and correspondence algorithm, but its usage for a real-life application with a customized design, with an architecture specifically for real-time processing and response.

Eventually, we aim to produce the device that will remove the necessity for a companion for visually impaired people and to that purpose, we dedicate this work to all who distressed by being unable on daily habits caused by visual barriers.

Architectural details and VHDL files,as well as simulation test-benches and implementation files, can be found on Wiki page and downloads section. This page will be updated incrementally, and any kind of support,feedback or recommendation is much appreciated.

Note: All design files shared so far include only the delay model,so that we can simulate the design. Synthesizable design shall be available after the delays are replaced with counters which ensure correct timing of the system.
