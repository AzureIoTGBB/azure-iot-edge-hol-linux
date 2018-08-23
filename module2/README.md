# Azure IoT Edge Hands On Labs - Module 2

Created and maintained by the Microsoft Azure IoT Global Black Belts

## KNOW BEFORE YOU START

This version of module2 uses a python script to act as our "IoT Device".  If you have trouble with it, or just prefer .NET, there is an alternate .NET Core version available [here](./README.dotnet.md)

## Introduction

For this step of the lab, we are going to create our "IoT Device".  Let's get started...

## Clone lab locally

We need some of the module2 files later in this lab.  So let's go ahead and clone this github repository locally

```bash
cd ~
git clone http://github.com/azureiotgbb/azure-iot-edge-hol-linux
```

## Create "IoT Device"

For our device, we will leverage a python script that emulates our IoT Device.  The device leverages our python Azure IoT SDK to connect to the hub and sends temperature and humidity data to the Edge in a 'sawtooth' pattern  (so we can later test going above and below the 'threshold')

### setup libraries and pre-requisites

* Because we are in public preview with IoT Edge, we need to leverage a preview version of the python SDK.  To install that preview version, we need to clone it (the modules-preview branch) build it.  To start, from your home folder (cd ~):

```bash
git clone --recursive http://github.com/azure/azure-iot-sdk-python

cd ~/azure-iot-sdk-python/build_all/linux

sudo ./setup.sh
```

After setup.sh completes, we need to build the SDK.  To do so, run

```bash
sudo ./build.sh
```

Once the build is done, we need to copy over the compiled library to our solution folder

```bash
cp ../../device/samples/iothub_client.so ~/azure-iot-edge-hol-linux/module2
```

## Set IoT Device connection details

We need to fill in a couple of pieces of information into our python script.

```bash
cd ~/azure-iot-edge-hol-linux/module2

nano iotdevice.py
```

* In the line below

```Python
connection_string = "<IoT Device connection string here>"
```

* put your "IoT Device connection string" inside the quotes.  Onto the end of your connection string, append __**";GatewayHostName=mygateway.local"**__.  This tells our Python script/device to connect to the specified IoTHub in it's connection string, but to do so __**through the specified Edge gateway**__

* save and close the script

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

After a couple of minutes, if you run 'docker ps' again, you should see a new module/container called "edgeHub' running.  This is the local IoT Hub-like engine that will store and forward our messages and act like a local IoTHub to our downstream devices

if you run 'docker logs -f edgeHub', you should see that the Hub has successfully created a route called "route' and is up and listening on port 8883. (the TLS port for MQTT locally)

The edge device is now ready for our device to connect.

We can confirm our edgeHub is set up and listening for connections from downstream "leaf" devices, we'll open a test TLS connection using the openssl tool.   From the bash prompt, run

```bash
openssl s_client -connect mygateway.local:8883 -CAfile ~/edge/certs/azure-iot-test-only.root.ca.cert.pem
```

At the bottom of the output, you should see the text "Verified (ok)".  Type a capital Q and hit enter to exit.


### Monitor our IoT Hub

If you don't already have one, open an additional putty connection to your Edge device (so we can run our python script in the other)

In that new window, to monitor the IoT Hub traffic, we will us the monitor-events function of iothub-explorer, like this:

```bash
iothub-explorer monitor-events <IoT Device Name> -r --login "<IoTHub iothubowner connection string>"
```

where \<IoT Device Name> is the name of your IoT device (as originally creatd in the Azure Portal and used in our python script) and \<IoTHub iothubowner connection string> is the IoT Hub-leve connection string retrieved and stored earlier in Notepad.  After running the command, it should say "Monitoring events from device \<IoT Device Name>..."  No events will be shown yet, as we haven't started our IoT Device yet

### start the local IoT device

Back on the edge device, run the following commands to 'run' our IoT device

```bash
cd ~/azure-iot-edge-hol-linux/module2

python -u iotdevice.py
```

You should see debug output indicating that the device was connected to the "IoT Hub" (in actuality it is connected to the edge device) and see it starting sending humidity and temperature messages.

Your output should look something like the below screenshot.  Note that the lines showing the data being sent are interspersed with confirmation messages that it was successfully sent.  If your output doesn't look like this (and is, for example, only showing the temp/humidity readings and no confirmation messages), then YOUR PYTHON SCRIPT IS NOT CORRECTLY WORKING WITH EDGE and you need to troubleshoot.

![python_success](/images/python_success.png)

### Observe D2C messages

In the iothub-explorer output window opened earlier, you should see messages flowing though the hub.  These messages have come from the device, to the local Edge Hub and been forwarded to the cloud based IoT Hub in a store-and-forward fashion (i.e. transparent gateway).  Please note that there may be some delay before you see data in the monitor.

hit CTRL-C to stop monitoring.

### test Direct Method call

Finally, we also want to test making a Direct Method call to our IoT Device.  Later, this functionality will allow us to respond to "high temperature" alerts by taking action on the device.  For now, we just want to test the connectivity to make sure that edgeHub is routing Direct Method calls propery to our device.  To test:

```
 iothub-explorer device-method <IoT Device> "ON" --login "<IoTHub iothubowner connection string>"
```

where \<IoT Device> is the name of your IoT device (as originally creatd in the Azure Portal  -- this is the name used for your python script!) and \<IoTHub iothubowner connection string> is the IoT Hub-leve connection string retrieved and stored earlier in Notepad.

You should see debug output in the python script that is our IoT Device indicating that a DM call was made.  This is a stand-in for whatever action we would want to take on our real device in the event of an "emergency" high temp alert.

* repeat the process above, sending "OFF" as the command to toggle the "alert" back off.

in the bash session runing your python script, hit CTRL-C to stop the script.

## Summary

The output of module is still the raw output of the device (in CSV format).  We've shown that we can connect a device through the edgeHub to IoT Hub in the cloud.  In the next few labs, we will add modules to re-format the data as JSON, as well as aggregate the data and identify and take local action on "high temperature" alerts.

To continue with Module 3, click [here](/module3)