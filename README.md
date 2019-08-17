# Integrating physical devices with IOTA — The IOTA payment terminal

The 9th part in a series of beginner tutorials on integrating physical devices with the IOTA protocol.

![img](https://miro.medium.com/max/700/1*Gq9shC9PGo7R4Raqp_ryOw.jpeg)

------

## Introduction

This is the 9th part in a series of beginner tutorials where we explore integrating physical devices with the IOTA protocol. This tutorial is the final part in a sequence of tutorials where we will try to replicate a traditional fiat based debit card payment solution with an IOTA based solution. In this tutorial we will be building a simple IOTA based payment terminal using some low cost off-the-shelf components.

Before you start building this project you should first take a look at the [6th](https://medium.com/coinmonks/integrating-physical-devices-with-iota-the-iota-debit-card-part-1-42dc1a05f18), [7th](https://medium.com/coinmonks/integrating-physical-devices-with-iota-the-iota-debit-card-part-2-1f073060ae1d) and [8th](https://medium.com/coinmonks/integrating-physical-devices-with-iota-the-iota-debit-card-part-3-bc0f03b8b2c9) tutorials in this series as they all provide the foundation for building the project in this tutorial.

------

## The Use Case

Now that the hotel owner has his functional IOTA debit card payment system up and running there is one final issue he needs to address. This issue is related to how his new IOTA debit card solution can be used in a practical end user payment scenario. Up until this point, interaction with the system has been based on a classic monitor and keyboard connected to the USB ports on our Raspberry PI. While this is fine for administrative tasks such as creating and issuing new IOTA debit cards, it may not be very practical as an end user payment interface. Imagine the waiter in the hotel restaurant having to drag around a large monitor and keyboard whenever a guest wants to pay for dinner. Or having a large monitor and keyboard connected to the swimming-pool locker. This would be both impractical and expensive. What we need is something like a portable payment terminal or some simple user interface that could be integrated in to the swimming-pool locker. Designing and building a slick and sexy portable payment terminal or human machine interface (HMI) is a little beyond the scope of this tutorial. However, using some low-cost off-the shelf components we should be able to build a prototype that could function as a proof-of-concept (PoC) for a future industrialized version.

------

## Components

The components you will need to build this project is as follows:

1. Raspberry PI
2. 12 Key — 4x3 Matrix — Membrane Type Keypad
3. 1602 LCD Display with I2C
4. RC522 RFID Reader/Writer module
5. LED, breadboard, some wires and a resistor

*Note!*
*All components except the keypad and 1602 LCD display have been discussed in previous tutorials so i will be skipping any further details on them in this tutorial.*

**The 12 Key — 4x3 Matrix — Membrane Type Keypad**
The *12 Key — 4x3 Matrix — Membrane Type Keypad* is a simple keypad that can be used as an input device to any project that requires numeric input. The keypad is popular among tinkerers for its low cost and ease of use. You should be able to get one of ebay for a couple of bucks.

![img](https://miro.medium.com/max/589/1*8hupH9Sz4jQ7LZ-F6d7Zxw.jpeg)

*Note!*
*You can also get these keypads in a 16 (4x4) key configuration. This may come in handy in cases where you need to execute other functions besides getting the numeric input. Notice that you will have to do some adjustments to the keypad library used in this tutorial if you choose to use the 16 key version.*

**The 1602 LCD Display with I2C**
The 1602 LCD Display is a very handy and compact display commonly used for projects that requires simple character and numeric based communication. The 1602 has an adjustable back-light feature that makes it function equally well in light and dark environments. I have chosen to use the I2C version of the 1602 LCD in this project. The I2C version of the 1602 LCD has an additional I2C module (LCM1602) attached to the back of the LCD that allows for serial communication. Using the I2C module, we only need a minimum number of GIO pins and connections to get the LCD up and running with the PI.

![img](https://miro.medium.com/max/600/1*p_I2FTtHnTIGyYRPnVCqxA.jpeg)

![img](https://miro.medium.com/max/487/1*H--G_cmEx_XEPp7czbKsiA.jpeg)

------

## Wiring the project

Use the following connection diagram to connect the various components used in this project.

![img](https://miro.medium.com/max/700/1*PGeeUnWjkq2Jlj3mHERaKw.png)

------

## Required Software and libraries

Before we start writing our Python code for this project we need to make sure that we have all the required software and libraries installed on our Raspberry PI. See previous tutorials for more information on installing the Raspbian OS, Python, Pyota and the RC522 RFID Reader/Writer library.

To get our 1602 LCD and Keypad to function correctly, we first need to add a couple of new Python libraries to our project.

To include these libraries in your project, simply download the [I2C_LCD_driver.py](https://gist.github.com/huggre/02045689aaeb153f2c4cf6573eb0bbcb) and [keypad.py](https://gist.github.com/huggre/1cf79a4e9848aa3cfd1f7aa8eb072be2) files and put them in the same folder where you installed the MFRC522-python library.

See [here](http://www.circuitbasics.com/raspberry-pi-i2c-lcd-set-up-and-programming) to learn more about using the different functions provided by the I2C_LCD_driver.py library.

*Note!
Before we can get the 1602 LCD working with I2C we have to make sure that I2C communication is enabled on the Raspberry PI. See* [*here*](http://www.circuitbasics.com/raspberry-pi-i2c-lcd-set-up-and-programming) *to learn more about enabling I2C on the Raspberry PI.*

*Note!*
*Depending on the version of Raspberry PI you are using you may have to do some changes to the I2CBUS and ADDRESS variables at the beginning of the I2C_LCD_driver.py file. For older versions of the PI you may have to change the I2CBUS variable to 0. You may also have to change the ADDRESS variable to the address used by your LCD. My I2C address is 27 so i’ll change the value to 0x27.*

------

## The Python code

The Python code used in this project is a variation of the code used in the [previous tutorial](https://medium.com/coinmonks/integrating-physical-devices-with-iota-the-iota-debit-card-part-3-bc0f03b8b2c9), only difference is that we will be getting user input from the keypad instead of a classical keyboard. We will also be replacing the monitor with the 1602 LCD for visual communication.

```python
# Imports some required libraries
import iota
from iota import Address
import RPi.GPIO as GPIO
import MFRC522
import signal
import time
import regex

# Import library for LCD1602 display 
import I2C_LCD_driver

# Import library for 4x3 keypad
from keypad import keypad

# Define display object
mylcd = I2C_LCD_driver.lcd()

# Define keypad object
kp = keypad()

# Varibles to control and collect keypad input
pos = 0
sumstring = ""
pincode = ""
pinmode = False
keypad_reading = True

# Setup O/I PIN's
LEDPIN=12
GPIO.setmode(GPIO.BOARD)
GPIO.setwarnings(False)
GPIO.setup(LEDPIN,GPIO.OUT)
GPIO.output(LEDPIN,GPIO.LOW)

# URL to IOTA fullnode used when interacting with the Tangle
iotaNode = "https://nodes.thetangle.org:443"
api = iota.Iota(iotaNode, "")

# Preparing hotel owner recieving address, replace with your own recieving address
hotel_address = b'GTZUHQSPRAQCTSQBZEEMLZPQUPAA9LPLGWCKFNEVKBINXEXZRACVKKKCYPWPKH9AWLGJHPLOZZOYTALAWOVSIJIYVZ'

# Some variables to control program flow 
continue_reading = True
transaction_confirmed = False
       
# Capture SIGINT for cleanup when the script is aborted
def end_read(signal,frame):
    global continue_reading
    print "Ctrl+C captured, ending read."
    continue_reading = False
    GPIO.cleanup()


# Function that reads the seed stored on the IOTA debit card
def read_seed():
    
    seed = ""
    seed = seed + read_block(8)
    seed = seed + read_block(9)
    seed = seed + read_block(10)
    seed = seed + read_block(12)
    seed = seed + read_block(13)
    seed = seed + read_block(14)
    
    # Return the first 81 characters of the retrieved seed
    return seed[0:81]

# Function to read single block from RFID tag
def read_block(blockID):

    status = MIFAREReader.MFRC522_Auth(MIFAREReader.PICC_AUTHENT1A, blockID, key, uid)
    
    if status == MIFAREReader.MI_OK:
        
        str_data = ""
        int_data=(MIFAREReader.MFRC522_Read(blockID))
        for number in int_data:
            str_data = str_data + chr(number)
        return str_data
        
    else:
        mylcd.lcd_clear()
        mylcd.lcd_display_string('Auth error......', 1)
        mylcd.lcd_display_string('Trans aborted...', 2)

# Function for checking address balance 
def checkbalance(hotel_address):
    
    address = Address(hotel_address)
    gb_result = api.get_balances([address])
    balance = gb_result['balances']
    return (balance[0])

# Function that blinks the LED
def blinkLED(blinks):
    for x in range(blinks):
        GPIO.output(LEDPIN,GPIO.HIGH)
        time.sleep(1)
        GPIO.output(LEDPIN,GPIO.LOW)
        time.sleep(1)

# Validate that PIN consist of 4 digit's
def validate_pin(pin):
    if pin == "" or regex.fullmatch("\d{4}", pin):
        return True
    else:
        return False

# Create a 16 element list inkl. new PIN to be written to autorization block
def make_pin(pin):
    if pin == "":
        pin_data = [255, 255, 255, 255, 255, 255, 255, 7, 128, 105, 255, 255, 255, 255, 255, 255]
    else:
        pin_data=[]
        pin_letter_list = list(pin)
        for letter in pin_letter_list:
            pin_data.append(ord(letter))
        pin_data = pin_data + [255, 255, 255, 7, 128, 105, 255, 255, 255, 255, 255, 255]

    return pin_data

# Get hotel owner address balance at startup
currentbalance = checkbalance(hotel_address)
lastbalance = currentbalance

# Hook the SIGINT
signal.signal(signal.SIGINT, end_read)

# Create an object of the class MFRC522
MIFAREReader = MFRC522.MFRC522()

# Show welcome message
mylcd.lcd_display_string('Number of Blinks', 1)

# Loop while getting keypad input
while keypad_reading:

    # Loop while waiting for a keypress
    digit = None
    while digit == None:
        digit = kp.getKey()
   
    # Manage keypad input
    if pinmode == False:
        if digit == '*':
            mylcd.lcd_display_string(" ", 2, pos-1)
            pos = pos -1
            sumstring = sumstring[:-1]
        elif digit == '#':
            blinks = int(sumstring)
            mylcd.lcd_clear()
            mylcd.lcd_display_string('PIN code:', 1)
            pinmode = True
            pos = 0
        else:
            mylcd.lcd_display_string(str(digit), 2, pos)
            pos = pos +1
            sumstring = sumstring + str(digit)
    else:
        if digit == '*':
            mylcd.lcd_display_string(" ", 2, pos-1)
            pos = pos -1
            pincode = pincode[:-1]
        elif digit == '#':
            
            keypad_reading = False
            
        else:
            if pos <= 3:
                mylcd.lcd_display_string('*', 2, pos)
                pos = pos +1
                pincode = pincode + str(digit)
 
    time.sleep(0.3)


# Show waiting for card message
mylcd.lcd_clear()
mylcd.lcd_display_string('Waiting for card', 1)

# This loop keeps checking for near by RFID tags. If one is found it will get the UID and authenticate
while continue_reading:
               
    # Scan for cards    
    (status,TagType) = MIFAREReader.MFRC522_Request(MIFAREReader.PICC_REQIDL)

    # If a card is found
    if status == MIFAREReader.MI_OK:
        mylcd.lcd_display_string('Card detected...', 2)
    
    # Get the UID of the card
    (status,uid) = MIFAREReader.MFRC522_Anticoll()

    # If we have the UID, continue
    if status == MIFAREReader.MI_OK:
   
        # Get authentication key
        key = make_pin(pincode)[0:6]
        
        # Select the scanned tag
        MIFAREReader.MFRC522_SelectTag(uid)
        
        # Get seed from IOTA debit card
        SeedSender=read_seed()
        
        # Stop reading/writing to RFID tag
        MIFAREReader.MFRC522_StopCrypto1()
                      
        # Create PyOTA object using seed from IOTA debit card
        api = iota.Iota(iotaNode, seed=SeedSender)
        
        # Display checking funds message
        mylcd.lcd_clear()
        mylcd.lcd_display_string('Checking funds..', 1)
        mylcd.lcd_display_string('Please wait.....', 2)
              
        # Get available funds from IOTA debit card seed
        card_balance = api.get_account_data(start=0, stop=None)
            
        balance = card_balance['balance']
        
        # Check if enough funds to pay for service
        if balance < blinks:
            mylcd.lcd_clear()
            mylcd.lcd_display_string('No funds........', 1)
            mylcd.lcd_display_string('Trans aborted...', 2)
            exit()
        
        # Create new transaction
        tx1 = iota.ProposedTransaction( address = iota.Address(hotel_address), message = None, tag = iota.Tag(b'HOTEL9IOTA'), value = blinks)

        # Display sending transaction message
        mylcd.lcd_clear()
        mylcd.lcd_display_string('Sending trans...', 1)
        mylcd.lcd_display_string('Please wait.....', 2)

        # Send transaction to tangle
        SentBundle = api.send_transfer(depth=3,transfers=[tx1], inputs=None, change_address=None, min_weight_magnitude=14, security_level=2)
                       
        # Display confirming transaction message
        mylcd.lcd_clear()
        mylcd.lcd_display_string('Confirming trans', 1)
        mylcd.lcd_display_string('Please wait.....', 2)
        
        # Loop executes every 10 seconds to checks if transaction is confirmed
        while transaction_confirmed == False:
            currentbalance = checkbalance(hotel_address)
            if currentbalance > lastbalance:
                mylcd.lcd_clear()
                # Display transaction confirmed message
                mylcd.lcd_display_string('Success!!!......', 1)
                mylcd.lcd_display_string('Trans confirmed.', 2)
                #print("\nTransaction is confirmed")
                blinkLED(blinks)
                transaction_confirmed = True
                continue_reading = False
            time.sleep(10)
```

You can download the source code from [here](https://gist.github.com/huggre/55aa9337628706a4c8c5d6fbcd97c55a)

------

## Running the project

To run the the project, you first need to save the code in the previous section as a text file in the same folder as where you installed the MFRC522-python library.

Notice that Python program files uses the .py extension, so let’s save the file as **iota_terminal.py** on the Raspberry PI.

To execute the program, simply start a new terminal window, navigate to the folder where you saved *iota_terminal.py* and type:

**python iota_terminal.py**

You should now see a message on the LCD asking for the amount of <blinks> you want to purchase. Use the numbers on the keypad to input the desired amount and press # when done. You may use the * key to make corrections.

Next you will be asked for the PIN code currently assigned to your IOTA debit card. Use the numbers on the keypad to input the 4 digit PIN code and press # when done. You may use the * key to make corrections.

Next you will be asked to hold your IOTA debit card close to the RFID reader. As soon as the card is detected by the RFID reader the IOTA payment transaction will be executed. Updates on the transaction process will be displayed on the LCD.

As soon as the transaction is confirmed by the IOTA tangle, the LED will start blinking according to the number blinks payed for.

------

## Donations

If you like this tutorial and want me to continue making others, feel free to make a small donation to the IOTA address shown below.

![img](https://miro.medium.com/max/400/1*kV_WUaltF4tbRRyqcz0DaA.png)

> GTZUHQSPRAQCTSQBZEEMLZPQUPAA9LPLGWCKFNEVKBINXEXZRACVKKKCYPWPKH9AWLGJHPLOZZOYTALAWOVSIJIYVZ
