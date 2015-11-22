# Introduction #

Following design models a 20 MHz to 160 MHz clock synchronizer circuit for data transmission between pixel buffer and controller blocks of the system.
Design can be generalized for different clock requirements,however, this core is specifically designed for our system.

Simulation file can also be found in the bottom of the documentation.

I believe comments are explaining the core in technical manner.


# Design #

```vhdl


-- Engineer: 		 Mahircan Gül
-- Create Date:    16:50:49 08/19/2014
-- Design Name: 	 FIFO Clock for crossing 20 MHz and 160 MHz clock domains.
-- Module Name:    fifo_synchronizer - Behavioral
-- Project Name: 	 Obstacle Detection Project
-- Description:
--FIFO clock synchronizer core is used for synchronizing 20 MHz clock for FIFO read buffer and 160 MHz FSM clock.
--Each byte received from FIFO buffer is pushed into queue in 4-Byte SRAM in core with 50 ns period.
--Dequeue operation is performed in 160 MHz frequency.



library IEEE;
use IEEE.STD_LOGIC_1164.ALL;
use IEEE.NUMERIC_STD.ALL;



entity fifo_synchronizer is


Port ( data_src : in  STD_LOGIC_VECTOR (7 downto 0);			--Source data(output from OV7725 FIFO read buffer)
clk_src : in  STD_LOGIC;										--Source clock(20 MHz)
data_dst : out  STD_LOGIC_VECTOR (7 downto 0);		--Output data(to controller core)
clk_dst : in  STD_LOGIC;										--Target core(160 MHz)
empty : out  STD_LOGIC;										--Queue empty flag
full : out  STD_LOGIC;										--Queue full flag
start : in STD_LOGIC);										--Enable signal
end fifo_synchronizer;






architecture Behavioral of fifo_synchronizer is

constant fifo_back_offset								: integer := 3;				--Rear of the queue.
constant FIFO_ACCESS_TIME 								: TIME := 15 ns;
constant DST_CLK_PERIOD 								: TIME := 6.25 ns;         --160 Mhz destination clock

type fifo_buffer_type is array (0 to 3) of STD_LOGIC_VECTOR (7 downto 0);	--SRAM with 4-byte length
signal fifo_buffer : fifo_buffer_type;													--Queue instantination
signal fifo_read_pointer : integer range 0 to 3 ;									--Read index
signal fifo_write_pointer : integer range 3 downto 0 ;							--Write index

signal sig_empty : std_logic ;
signal sig_full : std_logic ;


signal reg_data_dst : std_logic_vector(7 downto 0) ;								--Output data register



begin

full <= sig_full;
empty <= sig_empty;
data_dst <= reg_data_dst;



-- read_process description
-- This process samples the input data on rising edge of the source clock. As long as the buffer is empty, data is sampled and stored
-- in queue. When buffer is full, empty flag is deasserted and full flag becomes high. After that point, control is passed to write_process.
-- Data is started to be sampled again when buffer becomes empty again. Queue being empty or full indicates; either 4-bytes have been written
-- into buffer or there is no data on it,respectively.
read_process: process(clk_src)
begin
if rising_edge(clk_src) then
if start = '1' then
if(sig_full = '0' ) then
sig_empty <= '1';
fifo_buffer(fifo_read_pointer) <= data_src after FIFO_ACCESS_TIME;
if( fifo_read_pointer = fifo_back_offset) then
sig_empty <= '0' after FIFO_ACCESS_TIME;
fifo_read_pointer <= 0;
else
sig_empty <= '1';
fifo_read_pointer <= fifo_read_pointer + 1;
end if;
end if;
else
sig_empty <= '1';
end if;
end if;
end process;


-- write_process description
-- This process writes the data in queue into output data register with the rising edge of destination clock. Operation continues until
-- all 4-bytes in buffer have been written into output pins. After queue becomes empty, control is passed to read_process.

write_process: process(clk_dst)
begin

if rising_edge(clk_dst) then
if start = '1' then
if sig_empty = '0'  then
if reg_data_dst=fifo_buffer(fifo_back_offset) then
null;
else
reg_data_dst <= fifo_buffer(fifo_back_offset-fifo_write_pointer) ;
if fifo_write_pointer = 0 then
sig_full <= '0' after DST_CLK_PERIOD;
fifo_write_pointer <= fifo_back_offset;
else
fifo_write_pointer <= fifo_write_pointer - 1;
sig_full <= '1';
end if;
end if;
end if;
else
sig_full <= '0';
end if;
end if;


end process;





end Behavioral;
```

# Simulation #

```vhdl



LIBRARY ieee;
USE ieee.std_logic_1164.ALL;
USE ieee.numeric_std.ALL;

ENTITY fifo_sync_tb IS
END fifo_sync_tb;

ARCHITECTURE behavior OF fifo_sync_tb IS

-- Component Declaration for the Unit Under Test (UUT)

COMPONENT fifo_synchronizer
PORT ( data_src : in  STD_LOGIC_VECTOR (7 downto 0);
clk_src : in  STD_LOGIC;
data_dst : out  STD_LOGIC_VECTOR (7 downto 0);
clk_dst : in  STD_LOGIC;
empty : out  STD_LOGIC;
full : out  STD_LOGIC;
start : in STD_LOGIC);
END COMPONENT;


--Inputs
signal data_src : std_logic_vector(7 downto 0) := (others => '0');
signal clk_src : std_logic := '0';
signal clk_dst : std_logic := '0';
signal start : std_logic := '0';

--Outputs
signal data_dst : std_logic_vector(7 downto 0);
signal empty : std_logic;
signal full : std_logic;


-- Clock period definitions
constant clk_src_period : time := 50 ns;
constant clk_dst_period : time := 6.25 ns;

--Constant definitions
constant pixel_number: integer := 640*480*2;

BEGIN

-- Instantiate the Unit Under Test (UUT)
uut: fifo_synchronizer PORT MAP (
data_src => data_src,
clk_src => clk_src,
data_dst => data_dst,
clk_dst => clk_dst,
empty => empty,
full => full,
start => start
);

-- Clock process definitions
clk_src_process :process
begin
clk_src <= '0';
wait for clk_src_period/2;
clk_src <= '1';
wait for clk_src_period/2;
end process;

clk_dst_process :process
begin
clk_dst <= '0';
wait for clk_dst_period/2;
clk_dst <= '1';
wait for clk_dst_period/2;
end process;



stim_proc: process
begin

wait for 10*clk_src_period ;
start <= '1';

for i in 0 to pixel_number loop
data_src <= std_logic_vector(to_unsigned(i, data_src'length));
wait for clk_src_period;
end loop;



wait;
end process;

END;

```