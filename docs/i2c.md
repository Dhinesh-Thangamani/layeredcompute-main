# Overview

nter-Integrated Circuit is a famous synchronous serial communication protocol used to connect peripherals to microprocessor or microcontroller.

The I2C’s beauty lies in its simplicity: It only uses two lines to connect devices on a bus. one for serial data (SDA) and another for serial clock (SCL).




## Hardware logic – opendrain, Pullup resistor

I2C consists of two lines Serial Data (SDA) and Serial Clock (SCL) to connect all the devices in a bus.

I2C uses controller-target mechanism to communicate with each other device.

## Open Drain:

Open Drain is a digital logic that allows device to pull the line low to communicate and leave the line high in inactive state. This logic allows multiple devices connect to a bus without contention.

To pull the line to low, pull up resistor is connected between power source and SDA, SCL lines. The pull up resistor keeps the lines at high state when none of devices actively using the bus.


## Device Addressing

Since I2c bus architecture uses two lines to connect multiple devices, the controller needs a method to establish 1 on 1 communication with target device.

To identify each device, a unique number is assigned to them called Address.

7 bit Address (Standard)

7 bits are used to address devices in a bus.

Upto 128 devices could be connected to a bus. 16 of these are reserved for special purposes, hence 112 devices could be connected.

10 bit Address (Extended)

To increase the number of devices on a bus, 10 bit address mode has been supported. Upto 1024 devices could be connected in this mode. 2 bytes are used to identify device in 10 bit addressing mode.


## Operating Modes

I2C supports several operating modes to adapt different nature of devices and solution requirements. Based on the device speed, power consumption different modes are available.

 Standard speed Mode: upto 100 kHz speed

 Fast mode: upto 400 kHz

 Fast + mode: upto 1 MHz

 High Speed mode: upto 3.4 MHz are supported

## Speed & physical constraints:

I2c has limitation to expand the speed. It is due to opendrain – pull up resistor logic.

When the device drop the signal to low, it drops suddenly, but when the pullup resistor brings the signal high involves capacitor to charge. That's where the RC impedance limitation comes into consideration.


## Summary

For over four decades, I2c has remained a staple in the electronics industry due to its overall simplicity, low cost, and minimal PCB footprint.

Even in modern complex system architectures, there is always a need to low speed interface – such as tiny NVRAM, environmental sensors, real time clock and timers, etc.

Compare to other low speed serial protocols like SPI, eSPI, or UART, I2c stands out by allowing dozens of devices to share just two physical lines. Its combination of low pin count, flexible addressing, and simple design ensures that I2C remains a dominant & essential interface in hardware design today.