### Tutorial: Writing an I2C Driver
I2C (which can be pronounced as either "I squared C" or "I two C") is a type of low cost serial bus that is commonly used to connect peripheral electronic devices, such as a sensor, to a microcontroller (such as the REV Robotics Expansion Hub).  The _FIRST_ Tech Challenge software has built-in support for several commercially available sensors.  The _FIRST_ Tech Challenge software development kit (SDK) also lets advanced users write their own software driver to integrate an I2C device with the FTC Robot Controller app.

This tutorial describes how to integrate an off-the-shelf, I2C sensor into the _FIRST_ Tech Challenge Android control system.  This is and **advanced** topic and requires knowledge of advanced programming concepts.  

<p align="center"><img src="https://github.com/FIRST-Tech-Challenge/WikiSupport/blob/master/ftc_app/images/WritingI2C/Dryw.jpg" width="150"><p>

This document was written by Andryw Wade, who was the FIRST Tech Challenge summer engineering intern for 2017.  He is an alumnus of the _FIRST_ Tech Challenge and FTC Team 8923 (Swerve Robotics, Perpetual Velocity) from Woodinville, WA.

### Example Source Code

The source code files used in this document can be found at the following link:

https://github.com/FIRST-Tech-Challenge/WikiSupport/tree/master/SampleOpModes/java/i2cExample

### Introduction
The FTC SDK comes with built in support for many external sensors, but teams may find a sensor that they want to use that is not supported by the SDK. Analog and digital sensors are easy to program, but creating a driver for an I2C device can be very involved. Fortunately, the FTC SDK has made it rather simple to create an I2C device driver.

### Required Materials
* I2C sensor
    * Many are available from 3rd party sellers like Adafruit.
* Sensor datasheet
    * Datasheets provide all of the information you’ll need to use the sensor, and a lot of other potentially useful information. These are generally found on the seller’s website, and can often be found from a Google search.
* Wires, connectors, headers, soldering station, etc.
    * You’ll need to physically connect the device to an I2C port, which will often require soldering of header pins to the sensor, and creating appropriate connectors. Many will have these supplies in their own workshops, or can be purchased from hobby shops.

### Procedure Outline

Below is a basic outline of the process involved with getting an I2C sensor up and running. The rest of this document goes into thorough detail of each step.

* Hardware
    * Buy sensor
    * Wire and connect to I2C port on Rev module or Modern Robotics Device Interface Module
* Software
    * Create driver class
    * Extend I2CDeviceSynch class and add required methods
    * Add address and registers of sensor
    * Create write and read methods
    * Write user methods

### Wiring
For the purpose of giving instruction on how to create an I2C driver, this document shows the process to create a driver for Adafruit’s MCP9808 temperature sensor. While the practical use of this sensor may be limited for FTC use, it’s an example of a simple I2C sensor that will act as a guide to creating other I2C devices. The sensor can be found on Adafruit’s website: https://www.adafruit.com/product/1782

<p align="center"><img src="https://github.com/FIRST-Tech-Challenge/WikiSupport/blob/master/ftc_app/images/WritingI2C/adaMCP9808.jpg" width="200"><p>

Adafruit sells the sensor chip connected to a PCB along with a pin header. The sensor itself if actually the small black chip in the middle of the PCB with 8 pins coming out of the sides. Adafruit’s PCB includes pull-up/down resistors for some of the pins for the user, so there’s no need for you to add them. Once you receive the sensor, the header pins need to be soldered onto the chip. For those who are experienced with soldering, this will be simple. For those with less experience, the important things to remember are to make sure the pins are electrically connected to the PCB, and that solder is not shorting any of the pins to one another.

<p align="center"><img src="https://github.com/FIRST-Tech-Challenge/WikiSupport/blob/master/ftc_app/images/WritingI2C/MCP9808Soldered.jpg" width="200"><p>

Once the pins have been soldered, the next step is to create a cable to connect the sensor pins to either the Rev module or the Modern Robotics Device Interface Module. For now, the only pins we care about are Vdd (supply voltage), Gnd (ground), SCL (serial clock line), and SDA (serial data line). These 4 pins allow us to power the sensor and communicate over I2C. The other 4 pins on the sensor can be added later on if desired.

If you’re connecting the sensor to a Rev module, the Rev Expansion Hub Guide (http://www.revrobotics.com/content/docs/REV-31-1153-GS.pdf) includes diagram of the pin-outs in section 1.2 (page 2), including the I2C ports, which contain pins named similarly to the ones on the temperature sensor.

<p align="center"><img src="https://github.com/FIRST-Tech-Challenge/WikiSupport/blob/master/ftc_app/images/WritingI2C/i2cports.jpg" width="300"><p>

It can be seen on the Adafruit website that this sensor accepts source voltages from 2.7V-5.5V. The Rev modules use 3.3V, and the Interface Modules use 5V, so the sensor will work with both modules with a simple 4-wire cable with a 4-pin header connector on the sensor side. The only difference is that the cable connector on the module side will need to be different depending on whether you’re using the Rev module or Interface Module.

For the Rev module, the cable needs a 4-pin JST connector on the module side. To make things simple, we can cut one of the JST cables provided with the Rev kit in half, and attach the header connector to the other side.

<p align="center"><img src="https://github.com/FIRST-Tech-Challenge/WikiSupport/blob/master/ftc_app/images/WritingI2C/jstcable.jpg" width="250"><p>

If you’re instead using the MR Device Interface Module, the page on the MR website for the Interface Module (http://www.modernroboticsinc.com/core-device-interface-module-2) includes the pin-outs for the I2C bus.

<p align="center"><img src="https://github.com/FIRST-Tech-Challenge/WikiSupport/blob/master/ftc_app/images/WritingI2C/i2cportsMR.jpg" width="300"><p>

Instead of a JST connector, your cable will need to have a 4-pin header connector on each end. For this, it would probably be best to save the cables that come with Rev kit, and instead use something else.

### Writing the Driver
Now that we can connect the temperature sensor to an I2C port, so it’s time to write the driver for the sensor. It should be noted that in the software, there is no difference between the Rev and Modern Robotics hardware; the code will be exactly the same. There are some examples in the SDK that show how to write I2C drivers, such as the Modern Robotics Range Sensor. This can be found by going to the file in the samples folder of the SDK called SensorMRRangeSensor. Then in the OpMode where the sensor is declared, control click on ModernRoboticsI2CRangeSensor to see the driver class.

In short, what we need to do is write in some required I2C setup code, then add in the code to communicate with the specific I2C device and code to interpret the raw sensor data. If desired, you can simply copy that driver and modify it as necessary, but for the sake of being thorough, this document will create a driver from scratch to better explain everything.

First, create a new class to be the sensor driver. If you are familiar with library structures, you can choose to put this into a library folder, or you can simply add it to the TeamCode folder along with all of your OpModes. Give the class a name that identifies it, such as MCP9808. The class then needs to extend I2CDeviceSynchDevice like so:

```
package org.firstinspires.ftc.teamcode;

import com.qualcomm.robotcore.hardware.I2cDeviceSynch;
import com.qualcomm.robotcore.hardware.I2cDeviceSynchDevice;

public class MCP9808 extends I2cDeviceSynchDevice<I2cDeviceSynch>
{
}
```

The device synch class requires that we implement a few methods in this class. This is indicated by an error message saying you need to implement a method called getManufacturer(), so we can add that in. Because the manufacturer is Adafruit, we can just return that as shown below. Once that method is done, we get another error at the top saying we need to include a method called doInitialize(). We have to return a boolean indicating whether the initialization was successful. Right now, we don’t have a way to check for that, and we don’t yet need to initialize anything, so we can just return true for now. Another error message appears saying we need to include a method called getDeviceName(). This returns a string, so we can just create one ourselves with the name of the sensor as shown below. The exact phrasing isn’t important. The last error we get asks us to create a constructor for the class. Because we’re using the I2C device, we have to include parameters for a device client, and pass that back to the super method as shown:


```
public class MCP9808 extends I2cDeviceSynchDevice<I2cDeviceSynch>
{
    @Override
    public Manufacturer getManufacturer()
    {

        return Manufacturer.Adafruit;
    }

    @Override
    protected synchronized boolean doInitialize()
    {
        return true;
    }

    @Override
    public String getDeviceName()
    {

        return "Adafruit MCP9808 Temperature Sensor";
    }

    public MCP9808(I2cDeviceSynch deviceClient)
    {
        super(deviceClient, true);
    }
}
```
In the construction, we need to add a couple statements in order to communicate with the sensor. The sensor starts off disengaged, meaning that there isn’t any communication, which allows us to change things like the I2C address (more on that later) without causing issues. Once everything has been set up, we need to engage the sensor to start communicating. We also need to run the arming state callback method that deals with situations involving USB cables disconnecting and reconnecting:

```
public MCP9808(I2cDeviceSynch deviceClient)
{
    super(deviceClient, true);

    super.registerArmingStateCallback(false);
    this.deviceClient.engage();
}
```

The next thing we can do is create a way to configure this specific sensor in the Robot Controller along with the rest of the hardware. We do this by adding an annotation to the class called `I2cDeviceType`, which must be accompanied by an annotation called `DeviceProperties`. `DeviceProperties` has 2 strings that must be set: a name, and an XML tag. The name is what appears in the robot configuration menu, and should be something human readable. The XML tag is what identifies the sensor in the code, and needs to be different for every sensor. This needs to follow XML naming rules, so no spaces. Make sure the values you pick are descriptive, like so:

```
@I2cDeviceType
@DeviceProperties(name = "MCP9808 Temperature Sensor", xmlTag = "MCP9808")
public class MCP9808 extends I2cDeviceSynchDevice<I2cDeviceSynch>
```

Optionally, you can add parameters to `DeviceProperties` that set a description and/or limit the sensor to use with a particular type of control system (REV or Modern Robotics). The description should be insertable into a sentence that describes what the sensor is. Here is an example of `DeviceProperties` with both of these options used.

```
@DeviceProperties(description = "an MCP9808 temperature sensor", compatibleControlSystems = ControlSystem.REV_HUB, name = "MCP9808 Temperature Sensor", xmlTag = "MCP9808",)`
```

Now in the robot configuration menu, we can see that the sensor has been added to the list of I2C devices:

<p align="center"><img src="https://github.com/FIRST-Tech-Challenge/WikiSupport/blob/master/ftc_app/images/WritingI2C/MCP9808Config.jpg" width="400"><p>

Now we can start writing the code for the sensor. Much of what is done from here requires the sensor datasheet. It’s listed on Adafruit’s website on the sensor page, and it can also be found with a Google search, such as “MCP9808 datasheet.” https://cdn-shop.adafruit.com/datasheets/MCP9808.pdf

In order to communicate with the sensor, we need to know its I2C address. Every datasheet is different, but for this sensor, the address is listed in section 3.5 (page 11) of the datasheet, along with how to change the address.

<p align="center"><img src="https://github.com/FIRST-Tech-Challenge/WikiSupport/blob/master/ftc_app/images/WritingI2C/addresspins.jpg" width="400"><p>

It’s standard for I2C devices to have 7-bit addresses, with an additional bit that represents read/write mode. However, some manufacturers like Modern Robotics use 8-bit addresses, which you may notice when looking through source code. This sensor has a 7-bit address, so that’s what we will use. If you are unsure of what address format your sensor uses, assume it’s a 7-bit address.

If you want to have multiple I2C sensors on the same bus, they each need to have different addresses for communication to work properly. Some sensors, including this one, allow you to change the address if there are overlapping addresses. For this sensor, you can change the last 3 bits of the address by pulling the 3 address pins high or low. Because there are 3 pins, each with 2 possible states, this results in 2^3 or 8 possible addresses for the sensor. The PCB includes pull-down resistors for these pins, so we can just leave them low. This results in the binary address being 0011000. When dealing with the binary forms of numbers, we will be using hexadecimal, because it’s much easier to work with. There are useful online converters, such as http://www.binaryhexconverter.com. The above binary address results in 0x18 (the 0x prefix indicates a hexadecimal number).

Now that we know what the sensor address is, we need to put that into our driver class. We can indicate this as the default address, which assumes the address pins are pulled low. We also need to inform the device client of the address during construction:

```
    public MCP9808(I2cDeviceSynch deviceClient)
    {
        super(deviceClient, true);

        this.deviceClient.setI2cAddress(ADDRESS_I2C_DEFAULT);

        super.registerArmingStateCallback(false);
        this.deviceClient.engage();
    }
```



Next, let’s add the user accessible registers. A register is a physical place on the sensor chip that stores information. The user accessible registers are the ones that we can access through the I2C communication, which enable us to read and write data to the registers. For an example with this sensor, we could read the temperature measured by the sensor, or write to the configuration register to adjust how the sensor acts. Section 5.1 (page 16) of the datasheet contains information on all of the user accessible registers, including the register addresses and a description of what each is used for.

<p align="center"><img src="https://github.com/FIRST-Tech-Challenge/WikiSupport/blob/master/ftc_app/images/WritingI2C/accessibleregisters.jpg" width="400"><p>

We may not need to use all of the registers, but we may as well add them all into our code in case we want to add them later. It’s best to create an enum with a list of the registers and their addresses in our code. The enum can include an integer to represent the register address, as indicated by the datasheet. Again, they are given in binary, so we need to convert them to hexadecimal:

```
    public enum Register
    {
        CONFIGURATION(0x01),
        T_LIMIT_UPPER(0x02),
        T_LIMIT_LOWER(0x03),
        T_LIMIT_CRITICAL(0x04),
        TEMPERATURE(0x05),
        MANUFACTURER_ID(0x06),
        DEVICE_ID_REVISION(0x07),
        RESOLUTION(0x08);

        public int bVal;

        Register(int bVal)
        {
            this.bVal = bVal;
        }
    }
```

Now that we have the registers set up, we can add a few more things to the initialization. It takes a relatively long time to read each register individually, and is much easier to read them all at once. In the background, there is a thread that repeatedly reads all of the registers at once and stores them in a cache. Then in our code when we read the sensor register, we’re actually just reading from the cache. In order for this to work, we need to tell that thread which registers to continuously read by creating a read window. This requires us to inform it of the first and last registers available, so we can add a couple items to our enum called first and last. Then we can create a method to set the read window based on the first and last enum items, and add that method to the construction:

```
    public enum Register
    {
        FIRST(0),
        CONFIGURATION(0x01),
        T_LIMIT_UPPER(0x02),
        T_LIMIT_LOWER(0x03),
        T_LIMIT_CRITICAL(0x04),
        TEMPERATURE(0x05),
        MANUFACTURER_ID(0x06),
        DEVICE_ID_REVISION(0x07),
        RESOLUTION(0x08),
        LAST(RESOLUTION.bVal);

        public int bVal;

        Register(int bVal)
        {
            this.bVal = bVal;
        }
    }

    protected void setOptimalReadWindow()
    {
        // Sensor registers are read repeatedly and stored in a register. This method specifies the
        // registers and repeat read mode
        I2cDeviceSynch.ReadWindow readWindow = new I2cDeviceSynch.ReadWindow(
                Register.FIRST.bVal,
                Register.LAST.bVal - Register.FIRST.bVal + 1,
                I2cDeviceSynch.ReadMode.REPEAT);
        this.deviceClient.setReadWindow(readWindow);
    }

    public MCP9808(I2cDeviceSynch deviceClient)
    {
        super(deviceClient, true);

        this.setOptimalReadWindow();
        this.deviceClient.setI2cAddress(ADDRESS_I2C_DEFAULT);

        super.registerArmingStateCallback(false); // Deals with USB cables getting unplugged
        // Sensor starts off disengaged so we can change things like I2C address. Need to engage
        this.deviceClient.engage();
    }
```

Now we have just one more I2C thing to set up, which are the read and write methods. The device client includes methods to do this for us, so we don’t need to worry about any of the details of I2C communication. All of the registers on this sensor are 2 bytes long, which is known as a short data type (with the exception of the resolution register). A byte data type has a length of 1 byte, an integer has a length of 4 bytes, and a long has a length of 8 bytes. For the reading method, we’re going to utilize the read method of the device client object, which has us specify the number of bytes, in this case, 2. For the writing method, we’re going to utilize the write method of the device client, which does not require us to specify the number of bytes. However, the device client read and write methods use byte arrays, whereas we want to use shorts, so we can use the TypeConversion class to convert between each, like so:

```
    protected void writeShort(final Register reg, short value)
    {
        deviceClient.write(reg.bVal, TypeConversion.shortToByteArray(value));
    }

    protected short readShort(Register reg)
    {
        return TypeConversion.byteArrayToShort(deviceClient.read(reg.bVal, 2));
    }
```

### Communicating With Manufacturer ID Register
Now we can start to write the code to interact with the sensor using these methods. Let’s first create a method that reads the manufacturer ID. This is a good method to write first, because it’s not very complicated, and it helps us make sure that we’re communicating with the sensor. Before writing any code, we should look at the datasheet to see how the information is stored in the register, which is in section 5.1.4 (page 27):

<p align="center"><img src="https://github.com/FIRST-Tech-Challenge/WikiSupport/blob/master/ftc_app/images/WritingI2C/register5-5.jpg" width="600"><p>

As we can see, the register is 2 bytes long, and every bit represents the manufacturer ID. We can write a simple method to read this register and return the bytes as a short data type like so:

```
    public short getManufacturerIDRaw()
    {
        return readShort(Register.MANUFACTURER_ID);
    }
```

Now that we have some functionality with the sensor, we can add it to an OpMode to test it. We first need to declare the sensor like any other, then initialize it in runOpMode() before the waitForStart(). The hardware map has most of the commonly used sensors included with it, but it doesn’t have the MCP9808, so we need to get it using a different method. Below is the code to do so:

```
    private MCP9808 tempSensor;

    public void runOpMode() throws InterruptedException
    {
        tempSensor = hardwareMap.get(MCP9808.class, "tempSensor");
```

The hardware map call gives us an instance of the MCP9808 class, which creates a connection to the physical sensor and runs the initialization and construction methods that we wrote earlier. At this point, we can use any of the methods we’ve written for this sensor. Let’s add the manufacturer ID method to the telemetry to see what it gives us:

```
    telemetry.addData("Manufacturer ID", tempSensor.getManufacturerIDRaw());
```

Before running it, we should predict what it will give us. The datasheet says the ID is 0x0054. Since the telemetry will display the number in decimal, we can convert the hexadecimal number to decimal, which results in 84. Running the OpMode, we can see the returned ID number:


<p align="center"><img src="https://github.com/FIRST-Tech-Challenge/WikiSupport/blob/master/ftc_app/images/WritingI2C/manufacturerid.jpg" width="400"><p>

Now we can see that we’re communicating with the sensor, and we’re getting good data back. If at this point you are receiving 0, go back through the steps and make sure everything has been set up properly. Make sure all wires and connections are good and not broken, and the code has been written as described.

### Ambient Temperature Register
Once we have confirmed that the sensor is working properly, we can start to write the code for the main function of this sensor, which is measuring temperature. We can write a similar method to get the raw data from the ambient temperature register like so:

```
public short getTemperatureRaw()
{
   return readShort(Register.TEMPERATURE);
}
```

Unfortunately, we can’t just use the raw data as we did before, because the data is encoded differently. We need to look at the datasheet for the ambient temperature register for how to extract the measured temperature, which is in section 5.1.3 (page 24).

<p align="center"><img src="https://github.com/FIRST-Tech-Challenge/WikiSupport/blob/master/ftc_app/images/WritingI2C/register5-4.jpg" width="600"><p>

Unfortunately, the data is not a typical 8-bit or 16-bit number. First of all, the first 3 bits of this register deal with the alert feature of the sensor, which we don’t care about right now. The datasheet suggest that we mask out the first 3 bits, which gives us just the 13-bit temperature data. In order to do this, we need to make use of bitwise operators, which are explained well here: https://www.tutorialspoint.com/java/java_basic_operators.htm

Let’s have a look at the possible values that can be returned by the sensor. It’s a 13-bit signed number in two’s complement format. The 13-bit value is stored in a 16-bit number (short), and we just masked out the first 3 bits, so they are going to be 0. So if the register gives us a positive number, such as +5, here’s what we’d expect to get and what we’d receive from the sensor:

Expect:
0000 0000 0000 0101 = +5
Receive:
0000 0000 0000 0101 = +5

Thus, positive numbers will work fine, so now let’s look at an example of a negative number, such as -5:

Expect:
1111 1111 1111 1011 = -5
Receive:
0001 1111 1111 1011 = 8187

Because the first 3 bits have been set to 0, we don’t get a negative number like we expect. So in order to account for this, we can set the first 3 bits to all 1s or 0s depending on the sign of the number. We can check the sign of the number by masking out all of the data bits except for the sign bit. According to the datasheet, the sign bit is the 13th bit, so we mask out the other bits by using the bitwise AND operator with the binary number 0001 0000 0000 0000 (0x1000). If the result is equal to 0x1000, that means the bit is 1, thus the number is negative. To set the first 3 bits to 0, we can use the bitwise AND operator with the data and the binary number: 0001 1111 1111 1111 (0x1FFF). To set the first 3 bits to 1, we can use the bitwise OR operator with the data and the binary number: 1110 0000 0000 0000 (0xE000). These will both force the first 3 bits to be 0 or 1 without affecting the rest of the number.

At this point, we now have the number set up properly, but there’s one more thing to do. According to the datasheet, the least significant bit has a value of 2^-4, or 1/16, meaning the resolution of the sensor is 1/16 degrees Celsius. That means that the number we just set up is 16 times larger than the actual temperature. In order to fix this, we need to multiply the number by the value of the least significant bit, which in this case is 1/16. That gives us the actual temperature that the sensor is measuring in degrees Celsius. Below is the method that performs all of these calculations:

```
    public double getTemperature()
    {
        short dataRaw = getTemperatureRaw();

        // The first 3 bits are alert bits that we don't care about here. We need to force them to
        // be 0s or 1s if the number is positive or negative depending on the sign
        if((dataRaw & 0x1000) == 0x1000) // Negative
            dataRaw |= 0xE000;
        else // Positive
            dataRaw &= 0x1FFF;

        // Multiply by least significant bit (2^-4 = 1/16) to scale
        return dataRaw / 16.0;
    }
```

Now that the method is done, we can add it to the telemetry, and we can see that it’s returning reasonable values:

<p align="center"><img src="https://github.com/FIRST-Tech-Challenge/WikiSupport/blob/master/ftc_app/images/WritingI2C/temperaturereasonable.jpg" width="400"><p>

### Temperature Limit Register
Let’s look at another feature of the sensor, which is the alert feature. It allows the user to define 3 different temperature boundaries at which point an alert will be sent out. A practical application of this could be to stop something if the temperature gets too high/low. When the alert is triggered, the alert pin of the sensor gets pulled high, so this could be connected to a digital pin of the Rev module. Alternatively, the alert state can also be read through the I2C communication from the ambient temperature register, which we ignored earlier. We can use this to see specifically which alert was triggered.

Before we can make use of the alert feature, we need to inform the sensor of what we want the boundaries to be. This requires us to use the temperature limit registers, described in section 5.1.2 (page 22) of the datasheet.

<p align="center"><img src="https://github.com/FIRST-Tech-Challenge/WikiSupport/blob/master/ftc_app/images/WritingI2C/register5-3.jpg" width="600"><p>

As we can see, it accepts an 11-bit number in two’s complement format, so we need to create a method that takes some number and converts it into an appropriate 11-bit number. Because there are 3 limits that we can use, rather than writing the same code 3 times, we can write a single method that converts the temperature into the 11-bit number, and sends that to the register specified by the user.

The method that we create will take a number and a register as parameters. We will need to make sure that both are valid before trying to put the number into the register (for example, an excessively large number like 5000 can’t be used, and the register can’t be the manufacturer ID register). Because the register is 11-bit with a least significant bit of 2^-2, that means the range of acceptable values is -256 to +255.75 (can’t go all the way to +256 because of the two’s complement format). Below is a start to this method with the parameter checking in place:

```
    public void setTemperatureLimit(double limit, Register register)
    {
        // Make sure we're given valid register
        if(!(register == Register.T_LIMIT_LOWER || register == Register.T_LIMIT_UPPER || register == Register.T_LIMIT_CRITICAL))
            throw new IllegalArgumentException("Invalid temperature limit register!");

        // Register only accepts values ranging from -256 to 255.75
        if(limit > 255.75 || limit < -256.0)
            throw new IllegalArgumentException("Temperature limit out of bounds!");

     }
```

At this point, we can be sure that both parameters are valid, so we need to convert the temperature into an 11-bit number. We first need to divide by the least significant bit to scale the number for the register, which is 2^-2 or 1/4. We then want to typecast the number to an integer so we don’t have any extra digits. We also need to shift the number 2 bits to the left, because the least significant bit of the register isn’t at the end.

The register takes a signed 11-bit number in two’s complement format, so we again need to consider positive and negative numbers. The unused bits of the register will always have a value of 0, even if you try to write a 1 to them. Below are examples of what would be stored in the register when we try to give it positive and negative numbers:

Input +5 after shifting:
0000 0000 0010 1000
Actual register bits:
0000 0000 0010 1000
Expected register bits:
0000 0000 0010 1000

Input -5 after shifting:
1111 1111 1110 1100
Actual register bits:
0001 1111 1110 1100
Expected register bits:
0001 1111 1110 1100

As we can see, both positive and negative numbers will be stored as expected, so we don’t need to do anything else. Below is the final method that does all of this:

```
    public void setTemperatureLimit(double limit, Register register)
    {
        // Make sure we're given valid register
        if(!(register == Register.T_LIMIT_LOWER || register == Register.T_LIMIT_UPPER || register == Register.T_LIMIT_CRITICAL))
            throw new IllegalArgumentException("Invalid temperature limit register!");

        // Register only accepts values ranging from -256 to 255.75
        if(limit > 255.75 || limit < -256.0)
            throw new IllegalArgumentException("Temperature limit out of bounds!");

        // Divide by least significant bit (2^-2 = 1/4)
        short temp = (short) (limit * 4.0);

        // Register shifted by 2
        writeShort(register, (short) (temp << 2));
    }
```

We can also write another method that reads this register to ensure that we put in the correct number. Again, because we have 3 registers to check, we can use a single method with a register as a parameter like we did for the set method. Let’s create a method that gets the raw data from the register like we did for the ambient temperature:

```
    public short getTemperatureLimitRaw(Register register)
    {
        // Make sure we're given valid register
        if(!(register == Register.T_LIMIT_LOWER || register == Register.T_LIMIT_UPPER || register == Register.T_LIMIT_CRITICAL))
            throw new IllegalArgumentException("Invalid temperature limit register!");

        return readShort(register);
    }
```

Then we’ll make another method to do all of the calculations, which is very similar to the method that reads the ambient temperature with a few differences. Because the least significant bit of the register isn’t at the end, we need to shift the bits by 2 to the right. We then need to set the first 5 bits to 0s or 1s after checking the sign bit, then multiply by the least significant bit (2^-2 or 1/4). Below is the method to do this:

```
    public double getTemperatureLimit(Register register)
    {
        // Register is shifted by 2
        short dataRaw = (short) (getTemperatureLimitRaw(register) >> 2);

        // The first 5 bits need to be forced to be 0s or 1s depending on the sign of the number
        if((dataRaw & 0x0400) == 0x0400) // Negative
            dataRaw |= 0xF800;
        else // Positive
            dataRaw &= 0x07FF;

        // Multiply by least significant bit (2^-2 = 1/4) to scale
        return dataRaw / 4.0;
    }
```

Now in our OpMode, we can set the temperature boundaries for the alert function, so we need a way to know when an alert has been triggered. As shown in the datasheet in section 5.1.3 (page 24), the first 3 bits of the ambient temperature register show which flags have been triggered.

<p align="center"><img src="https://github.com/FIRST-Tech-Challenge/WikiSupport/blob/master/ftc_app/images/WritingI2C/register5-4.jpg" width="600"><p>

For this, we can create 3 separate methods for each flag, checking to see if that bit is enabled. Below is how we could write them:

```
    public boolean criticalLimitTriggered()
    {
        return (getTemperatureRaw() & 0x8000) == 0x8000;
    }

    public boolean upperLimitTriggered()
    {
        return (getTemperatureRaw() & 0x4000) == 0x4000;
    }

    public boolean lowerLimitTriggered()
    {
        return (getTemperatureRaw() & 0x2000) == 0x2000;
    }
```

Now in our OpMode, we can use these methods to check for when the temperature goes either below the lower limit, or above the upper or critical limit:

```
   tempSensor.setTemperatureLimit(24, MCP9808.Register.T_LIMIT_LOWER);
   tempSensor.setTemperatureLimit(26, MCP9808.Register.T_LIMIT_UPPER);
   tempSensor.setTemperatureLimit(25, MCP9808.Register.T_LIMIT_CRITICAL);
```
```
    telemetry.addData("Lower Limit", tempSensor.getTemperatureLimit(MCP9808.Register.T_LIMIT_LOWER));
    telemetry.addData("Lower Limit Triggered", tempSensor.lowerLimitTriggered());
    telemetry.addData("Upper Limit", tempSensor.getTemperatureLimit(MCP9808.Register.T_LIMIT_UPPER));
    telemetry.addData("Upper Limit Triggered", tempSensor.upperLimitTriggered());
    telemetry.addData("Critical Limit", tempSensor.getTemperatureLimit(MCP9808.Register.T_LIMIT_CRITICAL));
    telemetry.addData("Critical Limit Triggered", tempSensor.criticalLimitTriggered());
```

<p align="center"><img src="https://github.com/FIRST-Tech-Challenge/WikiSupport/blob/master/ftc_app/images/WritingI2C/temperaturelimits.jpg" width="400"><p>

As we can see, the lower limit is triggered because the temperature is below the limit, but the other 2 have not been triggered. If the temperature rises, the states change as expected.

### Configuration Register
The last register we’ll look at is the configuration register. This temperature sensor doesn’t require any configuration in order to return the temperature, but some other sensors need to be configured before they can function properly. Let’s look at what options are available in the configuration register in section 5.1.1 (page 18):

<p align="center"><img src="https://github.com/FIRST-Tech-Challenge/WikiSupport/blob/master/ftc_app/images/WritingI2C/register5-2.jpg" width="600"><p>

As we can see, there are lots of options available, and the descriptions of each are listed in the datasheet (not listed here, because it’s over a page long). It’s common to create an enum for each setting available in the configuration register, where each enum contains hexadecimal numbers representing the different options for that setting. This temperature sensor has a relatively large number of setting available, so we’ll just go over a couple examples, and the idea can be expanded on later. Let’s use the hysteresis and alert control settings.

The alert control bit is described in the datasheet below the configuration register table:

<p align="center"><img src="https://github.com/FIRST-Tech-Challenge/WikiSupport/blob/master/ftc_app/images/WritingI2C/bit3.jpg" width="700"><p>

As we can see, the alert pin on the sensor is disabled by default, meaning its state will never change when an alert is triggered. Let’s create an enum as described for this setting, where one option is to disable the pin, and the other is to enable. Each enum item needs the hexadecimal number representing the bit setting, so taking into account the bit being shifted over, disabled would be 0x0000 and enabled would be 0x0008. The enum would look like this:

```
    public enum AlertControl
    {
        ALERT_DISABLE(0x0000),
        ALERT_ENABLE(0x0008);

        public int bVal;

        AlertControl(int bVal)
        {
            this.bVal = bVal;
        }
    }
```

The hysteresis bits are described below the configuration register table:

<p align="center"><img src="https://github.com/FIRST-Tech-Challenge/WikiSupport/blob/master/ftc_app/images/WritingI2C/bit10-9.jpg" width="600"><p>

Sections 5.2.2 and 5.2.3 (page 30) describe the behavior in more detail:


<p align="center"><img src="https://github.com/FIRST-Tech-Challenge/WikiSupport/blob/master/ftc_app/images/WritingI2C/section5.2.2.jpg" width="400"><p>


In a nutshell, this means that the states of the alerts change once the temperature rises above the limit, and drops below the limit minus the hysteresis value. For example, if the hysteresis is set to 3 degrees and the upper limit is 26, that upper alert will be activated when the temperature goes above 26, and deactivated when it goes below 23. Let’s create the enum for this setting in the same way as the previous one:

```
    public enum Hysteresis
    {
        HYST_0(0x0000),
        HYST_1_5(0x0200),
        HYST_3(0x0400),
        HYST_6(0x0600);

        public int bVal;

        Hysteresis(int bVal)
        {
            this.bVal = bVal;
        }
    }
```

Now we have the ability to change these settings in the configuration register. We can change the configuration as often as we want, but it’s most common to do it once during initialization. So let’s add some code to the doInitialize() method that writes configuration settings to the configuration register. To do this, we can take the hexadecimal value from each setting and combine them with the bitwise OR operator like so:

```
    @Override
    protected synchronized boolean doInitialize()
    {
        int configSettings = Hysteresis.HYST_1_5.bVal | AlertControl.ALERT_ENABLE.bVal;

        writeShort(Register.CONFIGURATION, (short) configSettings);

        return true;
    }

```

Now that a configuration has been written, rather than simply returning true and assuming everything is good, we can read the configuration register and compare it with the settings that we intended to write. If they match, we can return true, and if not, return false:

```
    @Override
    protected synchronized boolean doInitialize()
    {
        int configSettings = Hysteresis.HYST_1_5.bVal | AlertControl.ALERT_ENABLE.bVal;

        writeShort(Register.CONFIGURATION, (short) configSettings);

        return readShort(Register.CONFIGURATION) == configSettings;
    }
```

One thing to note is that bit 4 is not actually a setting, it is a flag indicating if any of the alerts have been triggered. This means that if any of the alerts are triggered when we check the configuration, the values won’t match. To fix this, we can mask out that bit, then do the comparison:

```
    @Override
    protected synchronized boolean doInitialize()
    {
        int configSettings = Hysteresis.HYST_1_5.bVal | AlertControl.ALERT_ENABLE.bVal;

        writeShort(Register.CONFIGURATION, (short) configSettings);

        // Mask out alert signal bit, which we can't control
        return (readShort(Register.CONFIGURATION) & 0xFFEF) == configSettings;
    }
```

And now we have the code written to modify the settings in the configuration register. We haven’t yet added all of the options for each setting, but the process would continue in the same way for each: create an enum for that setting with each available option and appropriate hexadecimal code, then combine the desired setting with the others in the initialization.

### Advanced Configuration
The above way of setting up the configuration works, however all of the settings are hardcoded into the initialization method. This may work sufficiently for you, but you may want to have a way to choose different settings for different OpModes. For this, we can keep all the code we’ve written, but we need to extend a different class that uses a parameters object. This example is a copy of the first class with a different name:

```
public class MCP9808Params extends I2cDeviceSynchDeviceWithParameters<I2cDeviceSynch>
```

This presents us with a few errors, because we need to create a parameters class within the driver class:

```
public class MCP9808Params extends I2cDeviceSynchDeviceWithParameters<I2cDeviceSynch>
    public static class Parameters implements Cloneable
    {
    }
```

This parameters class then needs to be specified in the extended class and in the constructor super method like this:

```
public class MCP9808Params extends I2cDeviceSynchDeviceWithParameters<I2cDeviceSynch, MCP9808Params.Parameters>
    public MCP9808Params(I2cDeviceSynch deviceClient)
    {
        super(deviceClient, true, new Parameters());
```

Now that we have the parameters class, we need to add a few things to it. First, we’re going to need to be able to clone parameter objects later on, so we’ll add code to implement the cloneable interface and add a method to create clones:

```
    public static class Parameters implements Cloneable
    {
        public Parameters clone()
        {
            try
            {
                return (Parameters) super.clone();
            }
            catch(CloneNotSupportedException e)
            {
                throw new RuntimeException("Internal Error: Parameters not cloneable");
            }
        }
    }
```

Next, we need to add fields to the parameters class, which are the actual parameters that we’re going to use for the sensor configuration. There should be one for each available setting, which were the enums we created before. Each should be given a default setting to avoid null pointers. We can also add a field for the I2C address, which allows the user to more easily specify different I2C addresses:

```
    public static class Parameters implements Cloneable
    {
        I2cAddr i2cAddr = ADDRESS_I2C_DEFAULT;

        // All settings available
        Hysteresis hysteresis = Hysteresis.HYST_0;
        AlertControl alertControl = AlertControl.ALERT_DISABLE;

        public Parameters clone()
        {
            try
            {
                return (Parameters) super.clone();
            }
            catch(CloneNotSupportedException e)
            {
                throw new RuntimeException("Internal Error: Parameters not cloneable");
            }
        }
    }
```

Now at this point, we still have an error at the top of the driver class, because we need to add an initialization method called internalInitialize(). This method actually serves the same purpose as the doInitialize() method, so we can just rename it. This method also needs to accept a parameter object like this:

```
    @Override
    protected synchronized boolean internalInitialize(@NonNull Parameters params)
    {
        int configSettings = Hysteresis.HYST_1_5.bVal | AlertControl.ALERT_ENABLE.bVal;

        writeShort(Register.CONFIGURATION, (short) configSettings);

        // Mask out alert signal bit, which we can't control
        return (readShort(Register.CONFIGURATION) & 0xFFEF) == configSettings;
    }

```

This time, however, we want to use the settings from the parameters object that we’ve been given rather than hard coding the settings. And since we added an option to change the I2C address, we can specify that as well:

```
    @Override
    protected synchronized boolean internalInitialize(@NonNull Parameters params)
    {
        this.parameters = params.clone();

        deviceClient.setI2cAddress(params.i2cAddr);

        int configSettings = params.hysteresis.bVal | params.alertControl.bVal;

        writeShort(Register.CONFIGURATION, (short) configSettings);

        // Mask out alert signal bit, which we can't control
        return (readShort(Register.CONFIGURATION) & 0xFFEF) == configSettings;
    }

```

Those are all the changes we need to make to the driver class in order to add the parameters. It should also be noted that this example doesn’t include all of the settings available in the configuration register, but these can easily be added later by following the same process for the other settings. Now we need to adjust the OpMode to make use of the available options. This is done right after the sensor is grabbed from the hardware map. We create a new parameter object, and set each of the settings to what we want them to be. Then we run the sensor’s initialization method like so:

```
    tempSensor = hardwareMap.get(MCP9808Params.class, "tempSensor");

    MCP9808Params.Parameters parameters = new MCP9808Params.Parameters();
    parameters.hysteresis = MCP9808Params.Hysteresis.HYST_1_5;
    parameters.alertControl = MCP9808Params.AlertControl.ALERT_ENABLE;
    tempSensor.initialize(parameters);
```

Now we have a very easy way for each OpMode to specify different settings depending on what’s needed. Again, it may be sufficient to have the settings hard coded, but this provides a somewhat more elegant solution.

### Other Sensors
As stated before, the practical uses of this temperature sensor are limited for FTC use, but it does a good job of showing the basics of creating an I2C sensor driver and the intricate details associated with doing so. To recap, the basic outline of the process is as follows:
* Hardware
    * Buy sensor
    * Wire and connect to I2C port on Rev module
    * Software
* Create driver class
    * Extend I2CDeviceSynch class and add required methods
    * Add address and registers of sensor
    * Create write and read methods
    * Write user methods

One of the more difficult things to figure out is bit manipulation. Many sensors have the data in a non-typical format, so it’s often necessary to manipulate the data to get the actual measurements. This is best learned with practice.