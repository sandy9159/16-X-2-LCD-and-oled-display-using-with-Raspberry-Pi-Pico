# 16-X-2-LCD-and-oled-display-using-with-Raspberry-Pi-Pico



# 1 Raspberry Pi Pico with 16X2 LCD Display

![image](https://user-images.githubusercontent.com/19898602/145058261-e5a8b26f-c2fb-4ffe-8307-3aafbc5339dd.png)


Hello,

In this article we are going to interface Raspberry Pi Pico with 16X2 LCD Display If you are new to Raspberry Pi Pico and wants to get started then click here.
here you can learn the basics of how to start with raspberry Pi Pico.

Video of raspberry PI Pico & 16×2 LCD

https://youtu.be/c8NpsleXUP0

# Required Components

List of required components is given below.

> Raspberry Pi Pico

> 16*2 LCD Display with I2C

> Jumper Wires

> Bread Board


#  Schematic Diagram

Below is the schematic diagram for Raspberry Pi Pico with 16X2 LCD Display

![image](https://user-images.githubusercontent.com/19898602/145058520-68323372-25a2-4d3d-ba2c-b8644ba2dcfc.png)


Follow this schematic diagram and make connections.

GND Pin of display with GND Pin of Raspberry Pi Pico

Vcc Pin of display with 5V Pin of Raspberry Pi Pico

SCL Pin of display with GP27 Pin of Raspberry Pi Pico

SDA Pin of OLED display with GP26 Pin of Raspberry Pi Pico


# Coding

To write and upload program here i used Thonny IDE. You can download from here.

Here i write code in micropython.

There are three files for code. Two files are library files and another is a main code file.

First make library files.

Copy and paste library code and save in pico as same name as i saved.

Copy paste below code and save in raspberry pi pico as “lcd_api.py”.

`````
import time

class LcdApi:

    LCD_CLR = 0x01              # DB0: clear display
    LCD_HOME = 0x02             # DB1: return to home position

    LCD_ENTRY_MODE = 0x04       # DB2: set entry mode
    LCD_ENTRY_INC = 0x02        # --DB1: increment
    LCD_ENTRY_SHIFT = 0x01      # --DB0: shift

    LCD_ON_CTRL = 0x08          # DB3: turn lcd/cursor on
    LCD_ON_DISPLAY = 0x04       # --DB2: turn display on
    LCD_ON_CURSOR = 0x02        # --DB1: turn cursor on
    LCD_ON_BLINK = 0x01         # --DB0: blinking cursor

    LCD_MOVE = 0x10             # DB4: move cursor/display
    LCD_MOVE_DISP = 0x08        # --DB3: move display (0-> move cursor)
    LCD_MOVE_RIGHT = 0x04       # --DB2: move right (0-> left)

    LCD_FUNCTION = 0x20         # DB5: function set
    LCD_FUNCTION_8BIT = 0x10    # --DB4: set 8BIT mode (0->4BIT mode)
    LCD_FUNCTION_2LINES = 0x08  # --DB3: two lines (0->one line)
    LCD_FUNCTION_10DOTS = 0x04  # --DB2: 5x10 font (0->5x7 font)
    LCD_FUNCTION_RESET = 0x30   # See "Initializing by Instruction" section

    LCD_CGRAM = 0x40            # DB6: set CG RAM address
    LCD_DDRAM = 0x80            # DB7: set DD RAM address

    LCD_RS_CMD = 0
    LCD_RS_DATA = 1

    LCD_RW_WRITE = 0
    LCD_RW_READ = 1

    def __init__(self, num_lines, num_columns):
        self.num_lines = num_lines
        if self.num_lines > 4:
            self.num_lines = 4
        self.num_columns = num_columns
        if self.num_columns > 40:
            self.num_columns = 40
        self.cursor_x = 0
        self.cursor_y = 0
        self.implied_newline = False
        self.backlight = True
        self.display_off()
        self.backlight_on()
        self.clear()
        self.hal_write_command(self.LCD_ENTRY_MODE | self.LCD_ENTRY_INC)
        self.hide_cursor()
        self.display_on()

    def clear(self):
        """Clears the LCD display and moves the cursor to the top left
        corner.
        """
        self.hal_write_command(self.LCD_CLR)
        self.hal_write_command(self.LCD_HOME)
        self.cursor_x = 0
        self.cursor_y = 0

    def show_cursor(self):
        """Causes the cursor to be made visible."""
        self.hal_write_command(self.LCD_ON_CTRL | self.LCD_ON_DISPLAY |
                               self.LCD_ON_CURSOR)

    def hide_cursor(self):
        """Causes the cursor to be hidden."""
        self.hal_write_command(self.LCD_ON_CTRL | self.LCD_ON_DISPLAY)

    def blink_cursor_on(self):
        """Turns on the cursor, and makes it blink."""
        self.hal_write_command(self.LCD_ON_CTRL | self.LCD_ON_DISPLAY |
                               self.LCD_ON_CURSOR | self.LCD_ON_BLINK)

    def blink_cursor_off(self):
        """Turns on the cursor, and makes it no blink (i.e. be solid)."""
        self.hal_write_command(self.LCD_ON_CTRL | self.LCD_ON_DISPLAY |
                               self.LCD_ON_CURSOR)

    def display_on(self):
        """Turns on (i.e. unblanks) the LCD."""
        self.hal_write_command(self.LCD_ON_CTRL | self.LCD_ON_DISPLAY)

    def display_off(self):
        """Turns off (i.e. blanks) the LCD."""
        self.hal_write_command(self.LCD_ON_CTRL)

    def backlight_on(self):
        """Turns the backlight on.
        This isn't really an LCD command, but some modules have backlight
        controls, so this allows the hal to pass through the command.
        """
        self.backlight = True
        self.hal_backlight_on()

    def backlight_off(self):
        """Turns the backlight off.
        This isn't really an LCD command, but some modules have backlight
        controls, so this allows the hal to pass through the command.
        """
        self.backlight = False
        self.hal_backlight_off()

    def move_to(self, cursor_x, cursor_y):
        """Moves the cursor position to the indicated position. The cursor
        position is zero based (i.e. cursor_x == 0 indicates first column).
        """
        self.cursor_x = cursor_x
        self.cursor_y = cursor_y
        addr = cursor_x & 0x3f
        if cursor_y & 1:
            addr += 0x40    # Lines 1 & 3 add 0x40
        if cursor_y & 2:    # Lines 2 & 3 add number of columns
            addr += self.num_columns
        self.hal_write_command(self.LCD_DDRAM | addr)

    def putchar(self, char):
        """Writes the indicated character to the LCD at the current cursor
        position, and advances the cursor by one position.
        """
        if char == '\n':
            if self.implied_newline:
                # self.implied_newline means we advanced due to a wraparound,
                # so if we get a newline right after that we ignore it.
                pass
            else:
                self.cursor_x = self.num_columns
        else:
            self.hal_write_data(ord(char))
            self.cursor_x += 1
        if self.cursor_x >= self.num_columns:
            self.cursor_x = 0
            self.cursor_y += 1
            self.implied_newline = (char != '\n')
        if self.cursor_y >= self.num_lines:
            self.cursor_y = 0
        self.move_to(self.cursor_x, self.cursor_y)

    def putstr(self, string):
        """Write the indicated string to the LCD at the current cursor
        position and advances the cursor position appropriately.
        """
        for char in string:
            self.putchar(char)

    def custom_char(self, location, charmap):
        """Write a character to one of the 8 CGRAM locations, available
        as chr(0) through chr(7).
        """
        location &= 0x7
        self.hal_write_command(self.LCD_CGRAM | (location << 3))
        self.hal_sleep_us(40)
        for i in range(8):
            self.hal_write_data(charmap[i])
            self.hal_sleep_us(40)
        self.move_to(self.cursor_x, self.cursor_y)

    def hal_backlight_on(self):
        """Allows the hal layer to turn the backlight on.
        If desired, a derived HAL class will implement this function.
        """
        pass

    def hal_backlight_off(self):
        """Allows the hal layer to turn the backlight off.
        If desired, a derived HAL class will implement this function.
        """
        pass

    def hal_write_command(self, cmd):
        """Write a command to the LCD.
        It is expected that a derived HAL class will implement this
        function.
        """
        raise NotImplementedError

    def hal_write_data(self, data):
        """Write data to the LCD.
        It is expected that a derived HAL class will implement this
        function.
        """
        raise NotImplementedError

    def hal_sleep_us(self, usecs):
        """Sleep for some time (given in microseconds)."""
        time.sleep_us(usecs)
        
        
        
    
        
        
``````        
Before moving fuurther I would like to tell you something about PCB

Yes PCB are the heart of the electronics based project usually we hesitate to try custom PCB and opt to homemade solutions

like breadboard or Zero PCB earlier I also was in the same boat, I hesitate to try custom PCB my belief was they are much expensive.

but then I came to know about [JLCPCB.COM](https://jlcpcb.com/IAT) and I was totally surprised how low price PCB's are they offering 

there PCB quality is best in market, now I always go with PCB for my project and [JLCPCB.COM](https://jlcpcb.com/IAT) is my trusted 

PCB manufacturer, you can also try there PCB service for more details you can visit their website [JLCPCB.COM](https://jlcpcb.com/IAT)
You can also try there new purple colour for PCB without any extra cost.
If new user signup today from this link [JLCPCB](https://jlcpcb.com/IAT ) you will get 27$ coupon from [JLCPCB](https://jlcpcb.com/IAT ).
![image](https://user-images.githubusercontent.com/19898602/134336832-cb9953e9-02a6-4ff7-9d27-2caad10fe7c7.png)
![image](https://user-images.githubusercontent.com/19898602/130722577-c30b7b43-ea89-4847-9c6b-058f9fabeda3.png)![image](https://user-images.githubusercontent.com/19898602/130722585-b5268db1-5f17-428f-ba60-b823140f2a70.png)


# 2 Raspberry Pi Pico with OLED Display


![image](https://user-images.githubusercontent.com/19898602/145059103-77705d33-805b-4ca5-97b5-dde22ada5695.png)

Hello,

In this article we are going to interface OLED display with Raspberry Pi Pico

If you are new to Raspberry Pi Pico and wants to get started then click here.
here you can learn the basics of how to start with raspberry Pi Pico.

List of required components is given below.


# Required Components

> Raspberry Pi Pico

> OLED Display

> Jumper Wires

> Bread Board

# Schematic Diagram

![image](https://user-images.githubusercontent.com/19898602/145059325-c06472e6-4ccd-41b1-854a-8971963d9ee3.png)


Follow this schematic diagram and make connections.

GND Pin of display with GND Pin of Raspberry Pi Pico

Vcc Pin of display with 3.3V Pin of Raspberry Pi Pico

SCL Pin of display with GP9 Pin of Raspberry Pi Pico

SDA Pin of OLED display with GP8 Pin of Raspberry Pi Pico

# Coding

To write and upload program here i used Thonny IDE. You can download from here.

Here i write code in micropython.

There are two files for code. One is a library for OLED display and another is a main code file.

Main code is given below


````
import machine
import ssd1306
import time

oled = ssd1306.SSD1306_I2C(128, 64, machine.I2C(0))

oled.text("Hello everyone", 0, 0)
oled.show()
time.sleep(2)
oled.text("Welcome to my", 0, 10)
oled.text("channel", 0, 20)
oled.show()
time.sleep(2)
oled.text("Thank You", 0, 30)
oled.show()

`````

Save this code in Raspberry Pi Pico and name it main.py

Now press run button.




