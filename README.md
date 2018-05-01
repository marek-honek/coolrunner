library ieee;
use ieee.std_logic_1164.all;
use ieee.std_logic_unsigned.all;    
USE ieee.numeric_std.ALL;          

-- Input/output description
entity top is
    port (
        BTN0: in std_logic;         						--submit btn
        BTN1: in std_logic;							--reset btn
        SW0: in std_logic;          						--switch 0 to 1 while loading
        CLK: in std_logic;
        LED: out std_logic_vector(3 downto 0);
        D_POS: out std_logic_vector(3 downto 0);
        D_SEG: out std_logic_vector(6 downto 0)
    );
end top;

-- Internal structure description
architecture Behavioral of top is
    signal clk_5: std_logic := '0';
    signal tmp_5: std_logic_vector(7 downto 0) := x"00";    
    signal bin: std_logic_vector(31 downto 0) := x"00000000";			--main vector
    signal display: std_logic_vector(4 downto 0) := "00000";
    signal pos: std_logic_vector(3 downto 0) := "0111";				--position on 7seg display
    signal cnt: std_logic_vector(4 downto 0) := "00000";			--ensures shifts	
    signal delay: std_logic := '0';						--delay -> first shift, then addition	
    signal load: std_logic := '1';						--loading or computing and display
    signal value: std_logic := '0';						--value of loaded index
    signal w8: std_logic := '1';						--wait for another rising edge on bnt 1
    signal overload: std_logic := '0';						--loaded value is higher than 9999
    signal index: std_logic_vector(3 downto 0) := "0000";			--index of loaded value
begin
    -------------------
    -- clock divider --
    -------------------

    -- increment auxiliary counter every rising edge of CLK
    -- if you meet half a period of 5 ms invert clk_5

    process (CLK)
    begin
        if rising_edge(CLK) then
            tmp_5 <= tmp_5 + 1;
            if tmp_5 = x"16" then
                tmp_5 <= x"00";
                clk_5 <= not clk_5;
            end if;
        end if;
    end process;


    ------------------------------
    -- load BIN and display BCD --
    ------------------------------

    value <= SW0;

    process (clk_5)
    begin
        
        
        if rising_edge(clk_5) then


            if BTN1 = '0' then							--reset
                load <= '1';
                bin(31 downto 0) <= x"00000000";        
                cnt <= "00000";
                index <= x"0";
                overload <= '0';
            end if;
            
            
            if load = '1' and BTN1 = '1' then               			--condition BTN0 in previous cycle
                if BTN0 = '0' and w8 = '1' then             			--fulfill bin 
                    if index < "1111" then
                        index <= index + '1';
                    else
                        index <= "0000";
                        load <= '0' ;
                    end if;
                    bin(to_integer(unsigned(index))) <= value;    		--conversion to integer
                    w8 <= '0';
                end if;
                if BTN0 = '1' and w8='0' then
                    w8 <= '1';
                end if;
            end if;
                         
            
            if load = '0' and BTN1 = '1' then
                if cnt < "10000" and delay = '0' then  				--shift         	
                    bin(31 downto 1) <= bin(30 downto 0);      		
                    bin(0) <= '0';
                    delay <= '1';
                    cnt <= cnt + '1';
                end if;
                            
                if cnt < "10000" and delay = '1' then				--addition
                    if bin(31 downto 28) > "0100" then
                        bin(31 downto 28) <= bin(31 downto 28) + "0011";
                    end if;
                    if bin(27 downto 24) > "0100" then
                        bin(27 downto 24) <= bin(27 downto 24) + "0011";
                    end if;
                    if bin(23 downto 20) > "0100" then
                        bin(23 downto 20) <= bin(23 downto 20) + "0011"; 
                    end if;
                    if bin(19 downto 16) > "0100" then
                        bin(19 downto 16) <= bin(19 downto 16) + "0011";
                    end if;
                        delay <= '0';
                end if;
            end if;
                
            if bin > x"0000270F" and cnt="00000" then         		--bin value more than 9999
                    overload <= '1';
            end if;
        end if;
         
    end process;        
    
                
    ------------------------------------
    -- state to seven-segment display --
    ------------------------------------
    --      0
    --     ---
    --  5 |   | 1
    --     ---   <- 6
    --  4 |   | 2
    --     ---
    --      3
        with display select
        D_SEG <= "1111001" when "00001",
                 "0100100" when "00010",
                 "0110000" when "00011",
                 "0011001" when "00100",
                 "0010010" when "00101",
                 "0000010" when "00110",
                 "1111000" when "00111",
                 "0000000" when "01000",
                 "0010000" when "01001",
                 "0001000" when "01010",
                 "0000011" when "01011",
                 "1000110" when "01100",
                 "0100001" when "01101",
                 "0000110" when "01110",
                 "0001110" when "01111",
                 "1100011" when "10001",
                 "1000111" when "10010",
                 "0100011" when "10011",
                 "1000000" when others;
         
      process(clk_5)
        begin
            if rising_edge(clk_5) then						--exceeding the limit value 9999
                if overload='1' then
                    if  pos = "0111" then
                        pos <= "1011";
                        display <= "10001";
                    elsif pos = "1011" then
                        pos <= "1101";
                        display <="10010";
                    elsif pos = "1101" then
                        pos <= "1110";
                        display <= "10011";
                    elsif pos = "1110" then
                        pos <= "0111";
                        display <= "00000";
                    end if;
                end if;
                
                
                if load = '0' and overload = '0' then				--display of BCD value below the limit
                    if  pos = "0111" then
                        pos <= "1011";
                        display(3 downto 0) <= bin(27 downto 24);
                    elsif pos = "1011" then
                        pos <= "1101";
                        display(3 downto 0) <= bin(23 downto 20);
                    elsif pos = "1101" then
                        pos <= "1110";
                        display(3 downto 0) <= bin(19 downto 16);
                    elsif pos = "1110" then
                        pos <= "0111";
                        display(3 downto 0) <= bin(31 downto 28);
                    end if;
                end if;
                
                
                if load = '1' and overload = '0' then				--display of uploaded BIN values
                        display(4) <= '0';				
                    if  pos /= "1110" then
                        pos <= "1110";
                        display(0) <= value;
                        display(3 downto 1) <= "000";
                    elsif pos = "1110" then
                        pos <= "0111";
                        display(3 downto 0) <= index;
                    end if; 
                end if;
             end if;
     end process;

     D_POS <= pos;
    ----------
    -- LEDs --
    ----------
    
    LED(0) <= BTN0;
    LED(3) <= not SW0;

end Behavioral;
