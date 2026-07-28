# I2C: An Overview

I²C — Quick Brief

Two wires, both open-drain with external pull-ups:

SCL — clock, driven by the controller (master)
SDA — bidirectional data

Devices can only pull low; the pull-up resistor provides the high. That's what makes multi-master arbitration and clock stretching possible for free.
