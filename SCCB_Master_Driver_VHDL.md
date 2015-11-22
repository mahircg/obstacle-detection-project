# Introduction #

In order to read/write to internal registers of OV7725 module, SCCB interface is used, which is quite similar with industry standard I2C.

As far as I've seen, there isn't any VHDL or Verilog design for SCCB interface, and some people in forums claim that they could successively communicate with SCCB slaves with I2C master cores. EDK users might give it a try to interface SCCB with I2C master, before delving into modified version of it.

Since we are not using EDK, and I2C master core is a part of it, best option is to use an open source I2C design. For that purpose, thanks to Scott Larson, who is the designer of the core we found on EEWiki (related links can be found on References part),we modified the open source I2C master core. Differences between SCCB and I2C interfaces could be found with a simple web search.

Core is simulated with testbench, however it is not tested with actual hardware.

# Simulation #

https://obstacle-detection-project.googlecode.com/git/sccb_timing.JPG

```

LIBRARY ieee;
USE ieee.std_logic_1164.ALL;
 
-- Uncomment the following library declaration if using
-- arithmetic functions with Signed or Unsigned values
--USE ieee.numeric_std.ALL;
 
ENTITY sccb_tb IS
END sccb_tb;
 
ARCHITECTURE behavior OF sccb_tb IS 
 
    -- Component Declaration for the Unit Under Test (UUT)
 
    COMPONENT i2c_master
	 
	 GENERIC(
    input_clk : INTEGER  ; --input clock speed from user logic in Hz
    bus_clk   : INTEGER );   --speed the i2c bus (scl) will run at in Hz
	 
    PORT(
         clk : IN  std_logic;
         reset_n : IN  std_logic;
         ena : IN  std_logic;
         addr : IN  std_logic_vector(6 downto 0);
         rw : IN  std_logic;
         data_wr : IN  std_logic_vector(7 downto 0);
         reg_addr : IN  std_logic_vector(7 downto 0);
         busy : OUT  std_logic;
         data_rd : OUT  std_logic_vector(7 downto 0);
         ack_error : BUFFER  std_logic;
         sda : INOUT  std_logic;
         scl : INOUT  std_logic
        );
    END COMPONENT;
    

   --Inputs
   signal clk : std_logic := '0';
   signal reset_n : std_logic := '0';
   signal ena : std_logic := '0';
   signal addr : std_logic_vector(6 downto 0) := (others => '0');
   signal rw : std_logic := '0';
   signal data_wr : std_logic_vector(7 downto 0) := (others => '0');
   signal reg_addr : std_logic_vector(7 downto 0) := (others => '0');

	--BiDirs
   signal sda : std_logic;
   signal scl : std_logic;

 	--Outputs
   signal busy : std_logic;
   signal data_rd : std_logic_vector(7 downto 0);
   signal ack_error : std_logic;

   -- Clock period definitions
   constant clk_period : time := 10.412 ns;
 
BEGIN
 
	-- Instantiate the Unit Under Test (UUT)
   uut: i2c_master 
			generic map
			
			(
			input_clk => 96_000_000, --input clock speed from user logic in Hz
			bus_clk   => 100_000     --speed the i2c bus (scl) will run at in Hz
			)
			PORT MAP (
          clk => clk,
          reset_n => reset_n,
          ena => ena,
          addr => addr,
          rw => rw,
          data_wr => data_wr,
          reg_addr => reg_addr,
          busy => busy,
          data_rd => data_rd,
          ack_error => ack_error,
          sda => sda,
          scl => scl
        );

   -- Clock process definitions
   clk_process :process
   begin
		clk <= '0';
		wait for clk_period/2;
		clk <= '1';
		wait for clk_period/2;
   end process;
 

   -- Stimulus process
   stim_proc: process
   begin		
		reset_n <= '0';		-- reset the device
		wait for 10*clk_period;
		reset_n <= '1';			
		wait for clk_period;
		data_wr <= "00000110";		--drive data,register address and slave address inputs
		reg_addr <= x"12";
		addr <= "0100001";
		
		ena <= '1';						
		
		wait for 1000*clk_period;
		ena <= '0';

      wait;
   end process;

END;

```


# VHDL Code #

```
--------------------------------------------------------------------------------
--
--   FileName:         i2c_master_modified.vhd
--   Dependencies:     none
--   Design Software:  ISE Project Navigator 64-bit Version 14.7 
--
--   HDL CODE IS PROVIDED "AS IS."  DIGI-KEY EXPRESSLY DISCLAIMS ANY
--   WARRANTY OF ANY KIND, WHETHER EXPRESS OR IMPLIED, INCLUDING BUT NOT
--   LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY, FITNESS FOR A
--   PARTICULAR PURPOSE, OR NON-INFRINGEMENT. IN NO EVENT SHALL DIGI-KEY
--   BE LIABLE FOR ANY INCIDENTAL, SPECIAL, INDIRECT OR CONSEQUENTIAL
--   DAMAGES, LOST PROFITS OR LOST DATA, HARM TO YOUR EQUIPMENT, COST OF
--   PROCUREMENT OF SUBSTITUTE GOODS, TECHNOLOGY OR SERVICES, ANY CLAIMS
--   BY THIRD PARTIES (INCLUDING BUT NOT LIMITED TO ANY DEFENSE THEREOF),
--   ANY CLAIMS FOR INDEMNITY OR CONTRIBUTION, OR OTHER SIMILAR COSTS.
--
--   Version History
--   Version 1.0 11/1/2012 Scott Larson
--     Initial Public Release
--   Version 2.0 06/20/2014 Scott Larson
--     Added ability to interface with different slaves in the same transaction
--     Corrected ack_error bug where ack_error went 'Z' instead of '1' on error
--     Corrected timing of when ack_error signal clears
--   Version 2.0.1	06/08/2014 Mahircan Gül
--     Original design is modified for controlling SCCB slave devices. 
--     Modification includes adding 3-Phase Write Transmission property,as well as
--     a data register input to write data register address to slave.
--------------------------------------------------------------------------------

LIBRARY ieee;
USE ieee.std_logic_1164.all;
USE ieee.std_logic_unsigned.all;

ENTITY i2c_master IS
  GENERIC(
    input_clk : INTEGER  ; --input clock speed from user logic in Hz
    bus_clk   : INTEGER );   --speed the i2c bus (scl) will run at in Hz
  PORT(
    clk       :  IN      STD_LOGIC;                    --system clock
    reset_n   :  IN      STD_LOGIC;                    --active low reset
    ena       :  IN      STD_LOGIC;                    --latch in command
    addr      :  IN      STD_LOGIC_VECTOR(6 DOWNTO 0); --address of target slave
    rw        :  IN      STD_LOGIC;                    --'0' is write, '1' is read
    data_wr   :  IN      STD_LOGIC_VECTOR(7 DOWNTO 0); --data to write to slave
	 reg_addr  :  IN      STD_LOGIC_VECTOR(7 DOWNTO 0); --register address to read/write
    busy      :  OUT     STD_LOGIC;                    --indicates transaction in progress
    data_rd   :  OUT     STD_LOGIC_VECTOR(7 DOWNTO 0); --data read from slave
    ack_error :  BUFFER  STD_LOGIC;                    --flag if improper acknowledge from slave
    sda       :  INOUT   STD_LOGIC;                    --serial data output of i2c bus
    scl       :  INOUT   STD_LOGIC);                   --serial clock output of i2c bus
END i2c_master;

ARCHITECTURE logic OF i2c_master IS
  CONSTANT divider  :  INTEGER := (input_clk/bus_clk)/4; --number of clocks in 1/4 cycle of scl
  TYPE machine IS(ready, start, command, slv_ack1, wr, rd,wr_reg_addr, slv_ack2,slv_ack3, stop); --needed states
  SIGNAL  state     :  machine;                          --state machine
  SIGNAL  data_clk  :  STD_LOGIC;                        --clock edges for sda
  SIGNAL  scl_clk   :  STD_LOGIC;                        --constantly running internal scl
  SIGNAL  scl_ena   :  STD_LOGIC := '0';                 --enables internal scl to output
  SIGNAL  sda_int   :  STD_LOGIC := '1';                 --internal sda
  SIGNAL  sda_ena_n :  STD_LOGIC;                        --enables internal sda to output
  SIGNAL  addr_rw   :  STD_LOGIC_VECTOR(7 DOWNTO 0);     --latched in address and read/write
  SIGNAL  addr_reg   :  STD_LOGIC_VECTOR(7 DOWNTO 0);     --latched in address and read/write
  SIGNAL  data_tx   :  STD_LOGIC_VECTOR(7 DOWNTO 0);     --latched in data to write to slave
  SIGNAL  data_rx   :  STD_LOGIC_VECTOR(7 DOWNTO 0);     --data received from slave
  SIGNAL  bit_cnt   :  INTEGER RANGE 0 TO 7 := 7;        --tracks bit number in transaction
  SIGNAL  stretch   :  STD_LOGIC := '0';                 --identifies if slave is stretching scl
BEGIN

  --generate the timing for the bus clock (scl_clk) and the data clock (data_clk)
  PROCESS(clk, reset_n)
    VARIABLE count  :  INTEGER RANGE 0 TO divider*4;  --timing for clock generation
  BEGIN
    IF(reset_n = '0') THEN                --reset asserted
      stretch <= '0';
      count := 0;
    ELSIF(clk'EVENT AND clk = '1') THEN
      IF(count = divider*4-1) THEN        --end of timing cycle
        count := 0;                       --reset timer
      ELSIF(stretch = '0') THEN           --clock stretching from slave not detected
        count := count + 1;               --continue clock generation timing
      END IF;
      CASE count IS
        WHEN 0 TO divider-1 =>            --first 1/4 cycle of clocking
          scl_clk <= '0';
          data_clk <= '0';
        WHEN divider TO divider*2-1 =>    --second 1/4 cycle of clocking
          scl_clk <= '0';
          data_clk <= '1';
        WHEN divider*2 TO divider*3-1 =>  --third 1/4 cycle of clocking
          scl_clk <= '1';                 --release scl
          IF(scl = '0') THEN              --detect if slave is stretching clock
            stretch <= '1';
          ELSE
            stretch <= '0';
          END IF;
          data_clk <= '1';
        WHEN OTHERS =>                    --last 1/4 cycle of clocking
          scl_clk <= '1';
          data_clk <= '0';
      END CASE;
    END IF;
  END PROCESS;

  --state machine and writing to sda during scl low (data_clk rising edge)
  PROCESS(data_clk, reset_n)
  BEGIN
    IF(reset_n = '0') THEN                 --reset asserted
      state <= ready;                      --return to initial state
      busy <= '1';                         --indicate not available
      scl_ena <= '0';                      --sets scl high impedance
      sda_int <= '1';                      --sets sda high impedance
      bit_cnt <= 7;                        --restarts data bit counter
      data_rd <= "00000000";               --clear data read port
    ELSIF(data_clk'EVENT AND data_clk = '1') THEN
      CASE state IS
        WHEN ready =>                      --idle state
          IF(ena = '1') THEN               --transaction requested
            busy <= '1';                   --flag busy
            addr_rw <= addr & rw;          --collect requested slave address and command
            data_tx <= data_wr;            --collect requested data to write
				addr_reg <= reg_addr;
            state <= start;                --go to start bit
          ELSE                             --remain idle
            busy <= '0';                   --unflag busy
            state <= ready;                --remain idle
          END IF;
			 
        WHEN start =>                      --start bit of transaction
          busy <= '1';                     --resume busy if continuous mode
          scl_ena <= '1';                  --enable scl output
          sda_int <= addr_rw(bit_cnt);     --set first address bit to bus
          state <= command;                --go to command
			 
        WHEN command =>                    --address and command byte of transaction
          IF(bit_cnt = 0) THEN             --command transmit finished
            sda_int <= '1';                --release sda for slave acknowledge
            bit_cnt <= 7;                  --reset bit counter for "byte" states
            state <= slv_ack1;             --go to slave acknowledge (command)
          ELSE                             --next clock cycle of command state
            bit_cnt <= bit_cnt - 1;        --keep track of transaction bits
            sda_int <= addr_rw(bit_cnt-1); --write address/command bit to bus
            state <= command;              --continue with command
          END IF;
			 
        WHEN slv_ack1 =>                   --slave acknowledge bit (command)
          IF(addr_rw(0) = '0') THEN        --write command
            sda_int <= addr_reg(bit_cnt);   --write first bit of register address
            state <= wr_reg_addr;           --go to write register addres
          ELSE                             --read command
            sda_int <= '1';                --release sda from incoming data
            state <= rd;                   --go to read byte
          END IF;
			 
			WHEN wr_reg_addr =>              --write byte of transaction
          busy <= '1';                     --resume busy if continuous mode
          IF(bit_cnt = 0) THEN             --write byte transmit finished
            sda_int <= '1';                --release sda for slave acknowledge
            bit_cnt <= 7;                  --reset bit counter for "byte" states
            state <= slv_ack2;             --go to slave acknowledge 
          ELSE                             --next clock cycle of write state
            bit_cnt <= bit_cnt - 1;        --keep track of transaction bits
            sda_int <= addr_reg(bit_cnt-1); --write next bit of address register to bus
            state <= wr_reg_addr;                   --continue writing
          END IF;
			 
			WHEN slv_ack2 =>						                
			sda_int <= data_tx(bit_cnt);		--write first bit of data  
			state <= wr;							--go to write data 

			 
        WHEN wr =>                         --write byte of transaction
          busy <= '1';                     --resume busy if continuous mode
          IF(bit_cnt = 0) THEN             --write byte transmit finished
            sda_int <= '1';                --release sda for slave acknowledge
            bit_cnt <= 7;                  --reset bit counter for "byte" states
            state <= slv_ack3;             --go to slave acknowledge (write)
          ELSE                             --next clock cycle of write state
            bit_cnt <= bit_cnt - 1;        --keep track of transaction bits
            sda_int <= data_tx(bit_cnt-1); --write next bit to bus
            state <= wr;                   --continue writing
          END IF;
			 
        WHEN rd =>                         --read byte of transaction
          busy <= '1';                     --resume busy if continuous mode
          IF(bit_cnt = 0) THEN             --read byte receive finished
             IF(ena = '1' AND addr_rw = addr & rw) THEN  --continuing with another read at same address
              sda_int <= '0';              --acknowledge the byte has been received
            ELSE                           --stopping or continuing with a write
              sda_int <= '1';              --send a no-acknowledge (before stop or repeated start)
            END IF;
            bit_cnt <= 7;                  --reset bit counter for "byte" states
            data_rd <= data_rx;            --output received data
            state <= slv_ack3;             --go to master acknowledge
          ELSE                             --next clock cycle of read state
            bit_cnt <= bit_cnt - 1;        --keep track of transaction bits
            state <= rd;                   --continue reading
          END IF;
			 
        WHEN slv_ack3 =>                   --modified design does no continue write/read even though new address is provided
														 --instead, SM is switched to initial state, pending for new commands
            scl_ena <= '0';                --disable scl
            state <= stop;                 --go to stop bit

        WHEN stop =>                       --stop bit of transaction
          busy <= '0';                     --unflag busy
          state <= ready;                  --go to idle state
      END CASE;    
    END IF;

    --reading from sda during scl high (falling edge of data_clk)
    IF(reset_n = '0') THEN  --reset asserted
      ack_error <= '0';
    ELSIF(data_clk'EVENT AND data_clk = '0') THEN
      CASE state IS
        WHEN start =>                  
          IF(scl_ena = '0') THEN                  --starting new transaction
            ack_error <= '0';                     --reset acknowledge error output
          END IF;
        WHEN slv_ack1 =>                          --receiving slave acknowledge (command)
          IF(sda /= '0' OR ack_error = '1') THEN  --no-acknowledge or previous no-acknowledge
            ack_error <= '1';                     --set error output if no-acknowledge
          END IF;
        WHEN rd =>                                --receiving slave data
          data_rx(bit_cnt) <= sda;                --receive current slave data bit
        WHEN slv_ack2 =>                          --receiving slave acknowledge (write)
          IF(sda /= '0' OR ack_error = '1') THEN  --no-acknowledge or previous no-acknowledge
            ack_error <= '1';                     --set error output if no-acknowledge
          END IF;
        WHEN OTHERS =>
          NULL;
      END CASE;
    END IF;
    
  END PROCESS;  

  --set sda output
  WITH state SELECT
    sda_ena_n <=   data_clk WHEN start,  --generate start condition
              NOT data_clk WHEN stop,    --generate stop condition
              sda_int WHEN OTHERS;       --set to internal sda signal    
      
  --set scl and sda outputs
  scl <= '0' WHEN (scl_ena = '1' AND scl_clk = '0') ELSE 'Z';
  sda <= '0' WHEN sda_ena_n = '0' ELSE 'Z';
  
END logic;

```

# References #

[I2C Master Core Design by Scott Larson](http://www.eewiki.net/display/LOGIC/I2C+Master+(VHDL))

[SCCB Functional Specification](http://www.ovt.com/download_document.php?type=document&DID=63)