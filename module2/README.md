# Azure IoT Edge Hands On Labs - Module 2

Created and maintained by the Microsoft Azure IoT Global Black Belts

## Introduction

For this step of the lab, we are going to create our "IoT Device".  Let's get started...

## Create "IoT Device"

For our device, we will leverage a python script that emulates our IoT Device.  The device leverages our python Azure IoT SDK to connect to the hub and sends temperature and humidity data to the Edge in a 'sawtooth' pattern  (so we can later test going above and below the 'threshold')

### setup libraries and pre-requisites

* Because we are in public preview with IoT Edge, we need to leverage a preview version of the python SDK.  To install that preview version, we need to clone it (the modules-preview branch) build it.  To start, from your home folder (cd ~):

```bash
$ git clone --recursive -b modules-preview http://github.com/azure/azure-iot-sdk-python
$ cd ~/azure-iot-sdk-python/build_all/linux
$ sudo ./setup.sh
```

After setup.sh completes, we need to build the SDK.  To do so, run

```bash
$ sudo ./build.sh
```

Once the build is done, we need to copy over the compiled library to our solution folder

```bash
$ cp ../../device/device/samples/iothub_client.so ~/azure-iot-edge-hol-linux/module2
```
## create IoT Device in IoT Hub

* To represent our device in IoT Hub, we need to create an IoT Device
    * in the Azure portal, for your IoT Hub, click on "IoT Devices" in the left-nav  (note, this is different than the "IoT Edge Devices" we used previously)
    * click on "+Add Device" to add a new device.  Give the device a name and click "create"
    * capture (in notepad) the Connection String - Primary Key for your IoT device, we will need it in a moment

## set connection details

We need to fill in a couple of pieces of information into our python script.

* cd to the ~/azure-iot-edge-hol-linux/module2 folder

* Edit the script

```bash
nano readserial_nodevice.py
```

* In the line below

```Python
connection_string = "<connection string here>"
```

put your connection string in the quotes.  Onto the end of your connection string, append ";GatewayHostName=mygateway.local".  This tells our Python script/device to connect to the specified IoTHub in it's connection string, but to do so __**through the specified Edge gateway**__

Ok, we now have our device ready, so let's get it connected to the Hub

## Start IoT Edge, connect device, and see data flowing through

In this section, we will get the device created above connected to IoT Edge and see the data flowing though it.

* in the Azure portal, navigate to your IoT Hub, click on IoT Edge Devices (preview) on the left nav, click on your IoT Edge device
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

On your development box, in VS Code, click on the 'Extensions' tab on the left nav.  Search for an install the "Azure IoT Toolkit" by Microsoft.  Once installed (reload VS Code, if necessary), click back on the folder view and you should see a new section called "IOT HUB DEVICES".  Hover over it and you should see three dots "...".  Click on that and click "Set IoT Hub Connection String".  You should see an Edit box appear for you to enter a connection string.  Go back to notepad where we copied the connection strings earlier, and copy/paste the "IoT Hub level" (the 'iothubowner') connection string from earlier into the VS Code edit box and hit ok.

After a few seconds, a list of IoT Device should appear in that section.  Once it does, find the IoT Device (not the edge device) that is tied to your python script.  Right click on it and select "Start monitoring D2C messages".  This should open an output window in VS Code and show that it is listening for messages.

### start the local IoT device

Back on the edge device, run the following commands to 'run' our IoT device

```bash
$ cd ~/azure-iot-edge-hol-linux/module2
$ python -u readserial_nodevice.py
```

You should see debug output indicating that the device was connected to the "IoT Hub" (in actuality it is connected to the edge device) and see it starting sending humidity and temperature messages.

### Observe D2C messages

In the VS Code output window opened earlier, you should see messages flowing thought the hub.  These messages have come from the device, to the local Edge Hub and been forwarded to the cloud based IoT Hub in a store-and-forward fashion (i.e. transparent gateway).  Please note that there may be some delay before you see data in the monitor.

In VS Code, right click on your IoT Device and click on "Stop Monitoring D2C Messages".


### test Direct Method call

Finally, we also want to test making a Direct Method call to our IoT Device.  Later, this functionality will allow us to respond to "high temperature" alerts by taking action on the device.  For now, we just want to test the connectivity to make sure that edgeHub is routing Direct Method calls propery to our device.  To test:

* On the laptop, in VS Code, in the "IOT HUB DEVICES" section, right click on your IoT Device and click "Invoke Direct Method".
* in the edit box at the top for the method to call type "ON" (without the quotes) and hit \<enter>
* in the edit box for the payload, just hit \<enter>>, as we don't need a payload for our method

You should see debug output in the python script that is our IoT Device indicating that a DM call was made, and after a few seconds, the onboard LED on the device should light up.  This is a stand-in for whatever action we would want to take on our real device in the event of an "emergency" high temp alert.

* repeat the process above, sending "OFF" as the command to toggle the LED back off.

in the bash session runing your python script, hit CTRL-C to stop the script.

## Summary 

The output of module is still the raw output of the device (in CSV format).  We've shown that we can connect a device through the edgeHub to IoT Hub in the cloud.  In the next few labs, we will add modules to re-format the data as JSON, as well as aggregate the data and identify and take local action on "high temperature" alerts.