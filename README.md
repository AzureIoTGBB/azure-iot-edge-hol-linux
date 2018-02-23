# Azure IoT Edge Hands On Labs

Created and maintained by the Microsoft Azure IoT Global Black Belts



# NOTE - this is a work in progress..  It's not anywhere near done yet.  It is an alternate implementation of the HOLs used in the labs found [here](http://github.com/azureiotgbb/azure-iot-edge-hol)   :-)


## Overview

This hands-on lab demonstrates setting up, configuring, and developing modules for [Azure IoT Edge](https://azure.microsoft.com/en-us/services/iot-edge/).  The intent of these labs is not to cover exhaustively every IoT Edge topic, but rather cover a scenario that allows the student to learn and understand the basics of IoT Edge, develop modules and Edge ASA jobs, and perform Edge configuration, all in a pseudo-realistic use case.

These labs were originally developed to be delivered in-person by the Azure IoT GBBs to customers, however, they are available for any customers or partners to leverage, to play, or to learn.  Over time, they will evolve past this original use to incorporate other use cases.

In this workshop you will:

* Setup and configure a simple IoT Deviceto simply (and dumbly) send temperature over the serial port every 3 seconds.  This module will intentionally send the data in a "proprietary" format   (comma separated)
* create an IoT Edge module that read the simple CSV temp/humidity data from the device and converts to JSON and passes the message along
* create an Azure Stream Analytics module that a) aggregates the "every 3 seconds" data to a 30 second frequency to send to IoT Hub in the cloud and b) looks for temperatures above a certain threshold.  Then a threshold violation occurs, the module will drop an "alert" message on Edge
* create an IoT Edge module that reads the "alert" message from ASA and sends a Direct Method call to the IoT Device to turn ON or OFF an "alert"

The labs are broken up into the following modules:

* [Module 1](module1) - Prerequisites and IoT Edge setup
* [Module 2](module2) - Setup and program the "IoT Device"
* [Module 3](module3) - Develop "Formatter" module
* [Module 4](module4) - Azure Stream Analytics Edge job
* [Module 5](module5) - Develop "Alert" module

If you have questions or issues for the lab, check out our [troubleshooting](troubleshooting.md) guide!

Below is a conceptual flow for the labs to help visualize what is taking place and how the data is flowing through the system  ("T/H" is short for "temperature and humidity)

![conceptual drawing](/images/IoT-Edge-Labs-Conceptual-Design.png)

## Edge "Device"

For simplicity of setup, for our "Edge Device", we will actually be using an Ubuntu Linux VM hosted in Azure.  However, the principals are the same and the functionality is exactly the same as a Linux Edge Device on-premises would be.

__**Let's get started!!**__
