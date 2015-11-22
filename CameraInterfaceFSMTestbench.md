# Introduction #

Following simulation file describes the common behaviour of a OV7725 camera module along with FIFO buffer.

Testbench simulates an 8-bit counter as pixel byte data. It continously captures the frames and writes to FIFO.


# Testbench File #

```

LIBRARY ieee;
USE ieee.std_logic_1164.ALL;
 
 
ENTITY test2 IS
END test2;
 
ARCHITECTURE behavior OF test2 IS 
 
    -- Component Declaration for the Unit Under Test (UUT)
 
    COMPONENT CameraInterface
    Port ( 		vsync : in  STD_LOGIC;
				clk : in  STD_LOGIC;
				rclk : in  STD_LOGIC;
				start : in  STD_LOGIC;
				data_in : in  STD_LOGIC_VECTOR (7 downto 0);
				wrst : out  STD_LOGIC;
				wen : out  STD_LOGIC;
				rst : in  STD_LOGIC;
				rrst : out  STD_LOGIC;
				oe : out  STD_LOGIC;
				rck : out  STD_LOGIC;
				frame_rec : out  STD_LOGIC;
				data_out : out  STD_LOGIC_VECTOR (7 downto 0));
    END COMPONENT;
    
	constant half_clk_period 									: TIME := 5.206 ns;
	constant clk_period 										: TIME := 10.412 ns;
	constant max_count 											: INTEGER := 619617 ;
	
	constant fifo_period										: TIME := 83.296 ns;
	constant rclk_period										: TIME := 50  ns;
	constant half_rclk_period									: TIME := 25  ns;
	constant vsync_high_period 									: TIME := 522432.512 ns;			--4*784*2*83.3			
	constant vsync_low_period 									: TIME := 66087712.768 ns;			--506*784*2*83.3
	constant CYCLE_TIME_FIFO 									: TIME := 83.296 ns;
	constant CYCLE_TIME_INTERFACE 								: TIME := 10.412 ns;
	constant CYCLE_TIME_RCLK									: TIME := 50 ns;
	constant RESTART_DELAY 										: TIME := 0.1 ms;
	constant FIFO_ACCESS_TIME 									: TIME := 15 ns;
	constant VSYNC_HIGH_PERIOD_TIME 							: TIME := 522432.512 ns;			
	constant WRITE_DELAY 										: TIME := 2350946.304 ns;			
	constant WRITE_READ_START_DIFF 								: TIME := 32755902.112 ns;			
	constant READ_START 										: TIME := 35106848.416 ns;			
	constant READ_END 											: TIME := 65826848.416 ns;	
	constant VSYNC_RISING_EDGE 									: TIME := 66087712.768 ns;			
	constant READ_COMPLETE_TIME 								: TIME := 30720000 ns;			
	constant READ_START_TO_WRITE_DONE 							: TIME := 29935999.328 ns;			
	constant WRITE_READ_COMPLETE_DIFF 							: TIME := 784000.672 ns;			

   --Inputs
   signal vsync : std_logic := '0';
   signal clk : std_logic := '0';
   signal rclk : std_logic := '0';
   signal rclk_enable : std_logic := '0';
   signal start : std_logic := '0';
   signal data_in : std_logic_vector(7 downto 0) := (others => '0');
   signal rst : std_logic := '0';
   shared variable d1 : std_logic_vector(7 downto 0) := "00000001";
--	signal counter_enable : std_logic := '0';
--	signal counter_sig : std_logic := '0';
   shared variable count : INTEGER RANGE 0 TO max_count;
	
	
 	--Outputs
   signal wrst : std_logic;
   signal wen : std_logic;
   signal rck : std_logic;
   signal rrst : std_logic;
   signal oe : std_logic;
   signal frame_rec : std_logic;
   signal data_out : std_logic_vector(7 downto 0);
	



  
	
	
 
BEGIN
 
	-- Instantiate the Unit Under Test (UUT)
   uut: CameraInterface PORT MAP (
          vsync => vsync,
          clk => clk,
		  rclk => rclk,
          start => start,
          data_in => data_in,
          wrst => wrst,
          wen => wen,
          rst => rst,
          rrst => rrst,
          oe => oe,
		  rck => rck,
          frame_rec => frame_rec,
          data_out => data_out
        );

   -- Clock process definitions
   clk_process :process
   begin
		clk <= '0';
		wait for half_clk_period;
		clk <= '1';
		wait for half_clk_period;
   end process;
	
	rclk <= not rclk after half_rclk_period when rclk_enable = '1' else '0' ;
	

	


   -- Stimulus process
   stim_proc: process
   begin		

		
      rst <= '1';
      wait until  (wrst = '0' AND rrst = '0') ;			--2*CYCLE_TIME_FIFO
		rst <= '0';
		wait for 10*clk_period;									--10*CYCLE_TIME_FIFO
		start<='1';
		wait for 10*clk_period ;								--10*CYCLE_TIME_FIFO
		
		vsync <= '1';
		wait for vsync_high_period;							
		vsync <= '0';

		
		while true   loop
			if count = 0 then
				wait until oe = '0';
				rclk_enable <= '1';
			end if;

			wait for rclk_period ;
			d1 := to_stdlogicvector(to_bitvector(d1) rol 1);
			data_in <= d1;
			count := count + 1;
			if count = max_count then 
				vsync <= '1';
				wait for vsync_high_period;
				vsync <= '0';
				count := 0;
				rclk_enable<='0';
			end if;
			
			

			
		end loop;


   end process;

END;

```