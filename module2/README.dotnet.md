# Azure IoT Edge Hands On Labs - Module 2

Created and maintained by the Microsoft Azure IoT Global Black Belts

## KNOW BEFORE YOU START

This version of module2 uses a .NET Core app to act as our "IoT Device".  If you have trouble with it, or just prefer python, the default python version is available [here](./README.md)

## Introduction

For this step of the lab, we are going to create our "IoT Device".  Let's get started...

## Clone lab locally

We need some of the module2 files later in this lab.  So let's go ahead and clone this github repository locally

```bash
cd ~
git clone http://github.com/azureiotgbb/azure-iot-edge-hol-linux
```

## Create "IoT Device"

For our device, we will leverage a .NET Core that emulates our IoT Device.  The device leverages our C# Azure IoT SDK to connect to the hub and sends temperature and humidity data to the Edge in a 'sawtooth' pattern  (so we can later test going above and below the 'threshold')

### set connection details

We need to fill in a couple of pieces of information into our C# app.

```bash
cd ~/azure-iot-edge-hol-linux/module2/dotnet/readserial_nodevice
nano Program.cs
```

* In the line below

```CSharp
static string ConnectionString = "<IoT Device connection string>";  //don't forget the ;GatewayHostName=mygateway.local
```

* put your connection string in the quotes. Onto the end of your connection string, append __**";GatewayHostName=mygateway.local"**__.  This tells our .NET app/device to connect to the specified IoTHub in it's connection string, but to do so __**through the specified Edge gateway**__

hit CTRL-O, \<enter>, CTRL-X to exit

to ready our app, run

```bash
dotnet restore
dotnet build
```

Ok, we now have our device ready, so let's get it connected to the Hub

## Start IoT Edge, connect device, and see data flowing through

In this section, we will get the device created above connected to IoT Edge and see the data flowing though it.  Before we do it, we need to get the second part of the IoT Edge runtime, edgeHub, deployed.  The easiest way to do that is to go ahead and deploy a (temporarily) empty deployment

* in the Azure portal, navigate to your IoT Hub, click on IoT Edge on the left nav, click on your IoT Edge device you created earlier
* click on "Set Modules" in the top menu bar.  Later, we will add a customer module here, but for now, we are just going to set a route to route all data to IoT Hub, so click "Next"
* on the 'routes' page, make sure the following route is shown, if not, enter it.

```json
{
    "routes": {
        "route":"FROM /* INTO $upstream"
    }
}
```

click 'Next', and click 'Submit'

* $upstream is a special route destination that means "send to IoT Hub in the cloud".  So this route takes all messages (/*) and sends to the cloud.  This lets us, at this stage in the lab, confirm that Edge is working end-to-end before we move onto subsequent modules.

### confirm IoT Edge

The running instance of IoT Edge should have gotten a notification that it's configuration has changed.

If you run 'docker ps' again, you should see a new module/container called "edgeHub' running.  This is the local IoT Hub-like engine that will store and forward our messages and act like a local IoTHub to our downstream devices

if you run 'docker logs -f edgeHub', you should see that the Hub has successfully created a route called "route' and is up and listening on port 8883. (the TLS port for MQTT locally)

The edge device is now ready for our device to connect.

### Monitor our IoT Hub

We will monitor the messages that will be flowing from our IoT Device to IoT Hub using the azure CLI.  

* In the azure cloud CLI window opened earlier in your browser, run the following command.

```bash
az iot hub monitor-events -n [hub name]
```

where [hub name] is the short name of your IoT Hub you created earlier.  The screen should say "Starting event monitor, use ctrl-c to stop...".  No events will be shown yet, as we haven't started our IoT Device yet.  We will do that next.

### start the local IoT device

Back on the edge device, run the following commands to 'run' our IoT device

```bash
cd ~/azure-iot-edge-hol-linux/module2/dotnet/readserial_nodevice

dotnet run
```

You should see debug output indicating that the device was connected to the "IoT Hub" (in actuality it is connected to the edge device) and see it starting sending humidity and temperature messages.

### Observe D2C messages

In the azure cloud CLI window opened above, you should see messages flowing though the hub.  These messages have come from the device, to the local Edge Hub and been forwarded to the cloud based IoT Hub in a store-and-forward fashion (i.e. transparent gateway).  Please note that there may be some delay before you see data in the monitor.

hit CTRL-C to stop monitoring.

### test Direct Method call

Finally, we also want to test making a Direct Method call to our IoT Device.  Later, this functionality will allow us to respond to "high temperature" alerts by taking action on the device.  For now, we just want to test the connectivity to make sure that edgeHub is routing Direct Method calls propery to our device.  To test:

```bash
 az iot hub invoke-device-method -d [iot leaf device name] -mn ON -n [hub name]
```

where [iot leaf device name] is the name of your IoT device (as originally created in the Azure Portal  -- this is the name used for your python script!), NOT your edge device name and [hub name] is the short name of your IoT Hub you created earlier.

You should see debug output in the python script that is our IoT Device indicating that a DM call was made.  This is a stand-in for whatever action we would want to take on our real device in the event of an "emergency" high temp alert.

* repeat the process above, sending "OFF" as the command to toggle the "alert" back off.

in the bash session runing your .NET Core app, hit \<enter> to stop the script.

## Summary

The output of module is still the raw output of the device (in CSV format).  We've shown that we can connect a device through the edgeHub to IoT Hub in the cloud.  In the next few labs, we will add modules to re-format the data as JSON, as well as aggregate the data and identify and take local action on "high temperature" alerts.

To continue with Module 3, click [here](/module3)