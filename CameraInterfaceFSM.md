# Introduction #

Following design models the FSM for OV7725 camera interface. Design is tested using ISim and simulation file can be found in [CameraInterfaceFSMTestbench](CameraInterfaceFSMTestbench.md). Interface is not tested with actual hardware yet due to lack of VHDCI equipment(cables,breadboard,etc.) but timing ensures all the requirements of AL422 FIFO and OV7725 camera module.

Intended design would be integrated with a Microblaze PLB custom slave core for image acquisition. Data to be stored in memory via DMA for further processing. A custom PLB core is implemented to be used with Microblaze, however, since the changed design plan does not include usage of a soft CPU, custom PLB core will not be used.

Following core shall be used for image acquisition. It is connected to the control core, which handles data read/writes to/from external memory.

Camera device will be configured via SCCB interface to capture VGA frames.

Note: Synthesizable design shall be available after the delay models are actually implemented with counters.

# Timing #

Image data will be retrieved in 15 frames per second rate(640\*480@15 fps). In order to ensure this requirement, camera device will be targeted with a 12 MHz clock, regardless of the image format being used. Independence between frame format and internal clock (tPclk) could be verified as follows;

Time required to grab each frame (tGrab) <= 1/15 seconds

_**According to OV7725 datasheet**_

Time it takes to capture a frame (tCap) <= 480`*` tLine , tCap <= 0.0626 seconds

_**tCap ~= tGrab**_ shows that with an internal clock of 12 MHz, a frame rate of 15 fps could be achieved. Detailed explanation could be found at _References_ part.





| **Symbol** | **Parameter** | **Value** |
|:-----------|:--------------|:----------|
| tLine      | Period of one horizontal line | 784 `*` tP |
| tP         |  Period of one pixel             | 2`*`tPclk |
| tPclk      | Data period for 1 Byte | 83.296 ns  |
| tRclk      | Period of FIFO read clock | 20 ns     |

![https://obstacle-detection-project.googlecode.com/git/ov7725_timing_chart.jpg](https://obstacle-detection-project.googlecode.com/git/ov7725_timing_chart.jpg)

In order to prevent data duplication, read address counter of the FIFO buffer should start after write address counter, about 393.247 cycles(This information could be found on AL442 documentation).


Different clock speeds can be applied to write/read clock inputs of FIFO buffer. PCLK output of OV7725 is connected to WCLK(write clock) of FIFO. However, read clock of the buffer can be applied externally,and its value is quite crucial in order to ensure that the received data is valid. If control logic completes the read session while camera device still continues to write image data to buffer, received image will be incomplete or corrupted. Read and write operations can be done independently, as long as there is no address pointer conflict, so there is no need for race condition precautions. In this case, important question is; how to find the optimal read clock frequency for proper data acquisition.

OV7725 module has an internal PLL,which enables clock multiplication. By default, module PLL is enabled and x4 clock scale. Since we apply 12 MHz clock directly, we do not need any clock scaling(XCLK = PCLK). This is done by writing proper values to COM4 and CLKRC registers. Following values are written to the internal registers before applying power to the camera sensor.

_CLKRC <- 0100 0000_

_COM4  <- 0000 0000_

These values, along with COM7 register, are written via I2C master module to internal registers; initiated by controller(FSM).

Detailed information about register configuration will be available in controller documentation.



### Finding the right value for RCLK ###

Timing chart above shows that each frame is written into buffer and read from it within a VSYNC cycle(which has a duty cycle of 0.79%). It would have been unnecesary to try to find the optimal value for RCLK if there was any external HREF or HSYNC signal. Due to the absence of them, first we should be sure that the time difference between write and read start signals must be long enough to be invariant to the duty cycle of HREF (which has 81.6% duty cycle) signal.

Since there are 480 HREF signals for each frame, WRITE\_READ\_START\_DIFF must be greater than 479\*tP, so;

**_WRITE\_READ\_START\_DIFF_ > 479\*2\*tPclk**_is the first condition for data requirements._

Secondly, as mentione earlier, read operation must be completed after write operation is finalized. So second condition is;

_**START\_READ + 640\*480\*2\*tRclk > 498\*tLine**_

_**tRclk > 48.72 ns**_

Last condition for read clock is; read period should not exceed the VSYNC period, so read operation must be finished before falling edge of VSYNC signal.

_**START\_READ + 640\*480\*2\*tRclk < 510\*tLine**_

_**tRclk < 51.27 ns**_

Finally, we have the following inequality;

_**48.72 < tRclk < 51.27**_

By using it, it seems like using a clock of 20 MHz is fair enough to use as read clock of the buffer.

# ASM Diagram #
![https://obstacle-detection-project.googlecode.com/git/ov7725_interface.jpg](https://obstacle-detection-project.googlecode.com/git/ov7725_interface.jpg)


# RTL Design #

```vhdl


-- Engineer: Mahircan Gül
--
-- Create Date:    12:32:09 02/12/2014
-- Design Name:    OV7725 Camera Interface
-- Module Name:    CameraInterface - Behavioral
-- Project Name:   Obstacle Detection Project
-- Target Devices: Xilinx Spartan-6 LX45



library IEEE;
use IEEE.STD_LOGIC_1164.ALL;
USE ieee.std_logic_unsigned.all;



entity camera_interface is
Port ( vsync : in  STD_LOGIC;
clk : in  STD_LOGIC;
rclk : in  STD_LOGIC;
start : in  STD_LOGIC;
data_in : in  STD_LOGIC_VECTOR (7 downto 0);
wrst : out  STD_LOGIC;
wen : out  STD_LOGIC;
rst : in  STD_LOGIC;
rrst : out  STD_LOGIC;
oe : out  STD_LOGIC;
device_ready : out STD_LOGIC;
frame_rec : out  STD_LOGIC;
data_out : out  STD_LOGIC_VECTOR (7 downto 0));
end camera_interface;

architecture Behavioral of camera_interface is




constant CYCLE_TIME_FIFO 								: TIME := 83.296 ns;					-- Frequency of FIFO buffer -> 12 MHz
constant CYCLE_TIME_INTERFACE 						: TIME := 10.412 ns;					-- Frequency of camera interface -> 96 MHz
constant CYCLE_TIME_RCLK								: TIME := 50 ns;						-- Frequency of read clock input of FIFO -> 20 MHz
constant RESTART_DELAY 									: TIME := 100038.496 ns;			-- 100.000/CYCLE_TIME_FIFO~= 1200.5 , 1201*CYCLE_TIME_FIFO
constant FIFO_ACCESS_TIME 								: TIME := 15 ns;						-- Access timef or data output of FIFO
constant VSYNC_HIGH_PERIOD 							: TIME := 522432.512 ns;			-- VYSNC Logic-1 duration -> 4* tLine ( 4*784*2*CYCLE_TIME_FIFO)
constant WRITE_DELAY 									: TIME := 2350946.304 ns;			-- Start of rising_edge(HREF) -> 18*tLine
constant WRITE_READ_START_DIFF 						: TIME := 32755902.112 ns;			-- Difference between write&read address of FIFO( WRITE_READ_START_DIFF+393247*tPCLK)
constant READ_START 										: TIME := 35106848.416 ns;			-- Start time of read operation: WRITE_DELAY + WRITE_READ_DIFF
constant READ_COMPLETE_TIME 							: TIME := 30720000 ns;			   --	640*480*2*CYCLE_TIME_RCLK
constant READ_START_TO_WRITE_DONE 					: TIME := 29935999.328 ns;			-- Difference between read start and write end -> 498*tLine - READ_START
constant WRITE_READ_COMPLETE_DIFF 					: TIME := 784000.672 ns;			-- Difference between write end and read end -> READ_START + READ_COMPLETE_TIME - 498*tLine

type state_type is (rst_state,initialize,ready,wait_vsync_high,wait_vsync_low,enable_write,enable_read,write_done,read_done);
signal ps,ns : state_type;
signal  counter : STD_LOGIC;
signal  reg_data_in_enable : STD_LOGIC := '0';
signal  reg_data_out_enable : STD_LOGIC := '0';
signal  reg_pixel_data : STD_LOGIC_VECTOR (7 downto 0) := (others => '0');
signal  reg_wrst : STD_LOGIC := '1';
signal  reg_wen : STD_LOGIC := '1';
signal  reg_device_ready : STD_LOGIC := '1';
signal  reg_rrst : STD_LOGIC := '1';
signal  reg_oe : STD_LOGIC := '1';
signal  reg_frame_rec : STD_LOGIC := '0';



begin
sequential_proc: process(rst,clk)											-- Sequential logic with asynchronous reset
begin
if rst ='1' then
ps <= rst_state;
elsif rising_edge(clk)
then ps <=ns ;
else null ;
end if;
end process;



data_receive : process(rclk,reg_data_in_enable,data_in)		-- Data steering process.Input is read after access time, specified by AL442 FIFO documentation
begin
if rising_edge(rclk) then
if reg_data_in_enable = '1' then reg_pixel_data <= data_in after FIFO_ACCESS_TIME;
else reg_pixel_data <= (others => 'X');
end if;
end if;
end process;





comb_proc:		-- Combinational logic
process(ps,start,vsync)
begin
case ps is
when rst_state	 => 			reg_wrst <='1';
reg_rrst <='1' ;
reg_oe<='1';
reg_wen <= '1';
reg_frame_rec <= '0';
reg_data_in_enable <= '0';
reg_data_out_enable <= '0';
reg_device_ready <= '1';
ns <= initialize;

when initialize =>			reg_device_ready <= '1';
reg_wrst <='0' after RESTART_DELAY;
reg_rrst <='0' after RESTART_DELAY ;
reg_oe<='1';
reg_wen <= '1';
reg_frame_rec <= '0';
reg_data_in_enable <= '0';
reg_data_out_enable <= '0';
ns <= ready	after  RESTART_DELAY ;

when ready => 					reg_wen<='1';
reg_oe<='1';
reg_frame_rec <= '0';
reg_wrst <= '0';
reg_rrst <= '0';
reg_data_in_enable <= '0';
reg_data_out_enable <= '0';
reg_device_ready <= '0';
if start = '1' then ns <= wait_vsync_high;
else ns <= ready;
end if;

when wait_vsync_high => 	reg_wen<='1';
reg_oe<='1';
reg_frame_rec <= '0';
reg_wrst <= '0';
reg_rrst <= '0';
reg_data_in_enable <= '0';
reg_data_out_enable <= '0';
reg_device_ready <= '0';
if vsync = '1' then ns <= wait_vsync_low;
else ns <= wait_vsync_high;
end if;

when wait_vsync_low => 		reg_wen<='1';
reg_oe<='1';
reg_frame_rec <= '0';
reg_wrst <= '0';
reg_rrst <= '0';
reg_device_ready <= '0';
reg_data_in_enable <= '0';
reg_data_out_enable <= '0';
if vsync = '0' then
reg_wen <= '0' after WRITE_DELAY ;
reg_wrst <= '1' after WRITE_DELAY ;
ns <= enable_write after WRITE_DELAY;
else ns <= wait_vsync_low;
end if;


when enable_write  =>		reg_wrst <= '1';
reg_wen <= '0';
reg_rrst <= '0';
reg_device_ready <= '0';
reg_oe<='1';
reg_frame_rec <= '0';
reg_data_in_enable <= '0';
reg_data_out_enable <= '0';
reg_rrst <= '1' after WRITE_READ_START_DIFF;
reg_oe <= '0' after WRITE_READ_START_DIFF;
ns <= enable_read after WRITE_READ_START_DIFF ;

when enable_read =>			reg_wrst <= '1';
reg_rrst <= '1';
reg_wen <= '0' ;
reg_oe <= '0';
reg_device_ready <= '0';
reg_frame_rec <= '0';
reg_data_in_enable <= '1';
reg_data_out_enable <= '1';
reg_wrst <= '0' after READ_START_TO_WRITE_DONE;
reg_wen <= '1' after READ_START_TO_WRITE_DONE;
ns <= write_done after READ_START_TO_WRITE_DONE ;



when write_done 		=> 	reg_wrst <= '0';
reg_wen <= '1';
reg_data_in_enable <= '1';
reg_device_ready <= '0';
reg_data_out_enable <= '1';
reg_oe <= '1' after WRITE_READ_COMPLETE_DIFF;
reg_rrst <= '0' after WRITE_READ_COMPLETE_DIFF;
reg_frame_rec <= '1' after WRITE_READ_COMPLETE_DIFF;
ns <= read_done after WRITE_READ_COMPLETE_DIFF;

when read_done 		=> 	reg_wrst <= '0';
reg_frame_rec <= '1';
reg_rrst <= '0';
reg_wen <= '1';
reg_device_ready <= '0';
reg_oe <= '1';
reg_data_in_enable <= '0';
reg_data_out_enable <= '0';
ns <= wait_vsync_high;


when others 			=>		reg_wen<='1';
reg_oe<='1';
reg_frame_rec <= '0';
reg_wrst <= '0';
reg_rrst <= '0';
reg_data_in_enable <= '0';
reg_data_out_enable <= '0';
reg_device_ready <= '1';

end case;
end process;


frame_rec			<= reg_frame_rec;
data_out 			<= reg_pixel_data when reg_data_out_enable = '1' else (others => 'Z');
wrst 					<= reg_wrst;
wen 					<= reg_wen;
rrst 					<= reg_rrst;
oe 					<= reg_oe;
device_ready <= reg_device_ready;


end Behavioral;

```


# References and Resources #

[OV7725 Module Datasheet](http://www.uctronics.com/download/cam_module/CCM7725.zip)

[OV7725 Driver for STM32F4 MCU](http://embeddedprogrammer.blogspot.com.tr/2012/07/hacking-ov7670-camera-module-sccb-cheat.html)