# Azure IoT Edge Hands On Labs - Module 5

Created and maintained by the Microsoft Azure IoT Global Black Belts


# NOTE:   Please note - there is a temporary issue with IoT Edge resulting in a certificate validation problem.  Module 5 will currently not work

## Introduction

For this step of the lab, we are going to create an IoT Edge module that will respond to the "High Temperature Alert" message generated from our Azure Stream Analytics on the Edge module in module 4. This module will take that alert, and send a Direct Method call to our device to tell it to turn on or off a "high temperature" alert LED.  This is a simple example, but demonstrates communicating back to a device from a module.  More importantly, along with module 4, it shows an example of how to take local "action" on the Edge in situations where you need low latency processing and action on the edge.

We will develop our module in C# using .NET Core.  In this version of the lab, rather than having a development machine set up (and having to go through the pre-reqs for that), we will do the work entirely on our Linux VM running our Edge box.

We are primarily using C# and .NET Core for this lab because it is the most completely 'ready' language in terms of tooling at the time the labs are being written (during public preview).  However, by General Availability time for IoT Edge, you will be able to develop IoT Edge modules in any supported Azure IoT language, including C, C#, Java, Node, and Python.

## Developing our module

### Ensuring pre-requisites

You should have already installed the pre-requisites for this module earlier in module 3.  If not, switch over to that module and work down through the "install the IoT Edge project template" section.

### Creating the module "skeleton"

This process will be very similar to the actions taken in Module 3

```bash
cd ~/edge
dotnet new aziotedgemodule -n alertmodule
```

This will create a folder under edge called 'alertmodule'.  A number of nice things were generated for us, including a skeleton implementation of an IoT Edge Module.

### Modify the sample implementation

Now lets modify the sample code to implement our Alert module.

* in addition to the "Device Client" that other modules use, the Alerts sample also needs to use the "Service Client" to send Direct Method calls.  To add the service client, do the following:

```bash
cd ~/edge/alertmodule

dotnet add package Microsoft.Azure.Devices --version 1.16.0-preview-003

dotnet restore
```

Now open Program.cs to edit   (nano Program.cs)

* in the "using" section above the Program class, add the two following 'using' statements

```CSharp
using Newtonsoft.Json;
using Microsoft.Azure.Devices;
using Message = Microsoft.Azure.Devices.Client.Message;
using TransportType = Microsoft.Azure.Devices.Client.TransportType;
```

* in Program.cs, above the "Program" class, add the C# class that will represent our message we want to publish to the Hub

```CSharp
    class Alert
    {
        public string deviceID { get; set; }
        public string tempState { get; set; }
        public float temperature { get; set; }
        public DateTime eventdatetime { get; set; }
    }
```

* at the top of the "Program" class (just below the line that reads "static int counter"), add the following code

```CSharp
static ServiceClient ioTServiceClient;
```

* in the Main function, either delete or comment out this line of code:

```CSharp
if (!bypassCertVerification) InstallCert();
```

* In the Init function, right below the call to DeviceClient.CreateFromConnectionString, add this call

```CSharp
ioTServiceClient = ServiceClient.CreateFromConnectionString(connectionString);
Console.Writeline("Service client successfully initialized");
```

* the balance of our work will be in the PipeMessage function.  the top part of the function, which gets the "Device Client" instance that is stored in Context and makes sure it's valid, and that opens the message and gets it's content as a string, is boiler-plate and is fine for our purposes.  

The rest of the work we will be doing is inside the "if" block below (which checks to make sure we don't have an empty message):

```CSharp
    if (!string.IsNullOrEmpty(messageString))
    {
    }
```

* replace the code within the if block above, with the below code (note, CTRL-K cuts entire lines from nano, and you can copy the code from the page here and paste it by putting your cursor in the right place in Program.cs and right-clicking in putty)

```CSharp
string command = "";

Console.WriteLine($"Received message, body: [{messageString}]");

var alertMsg = JsonConvert.DeserializeObject<Alert[]>(messageString)[0];

if(alertMsg.tempState == "HIGH")
    command = "ON";
else
    command = "OFF";

Console.WriteLine($"Invoking Direct Method to {alertMsg.deviceID} for tempState={alertMsg.tempState}");

try {
    var result = await ioTServiceClient.InvokeDeviceMethodAsync(alertMsg.deviceID, new CloudToDeviceMethod(command));
    Console.WriteLine($"Result status: {result.Status}");
}
catch(Exception e)
{
    Console.WriteLine($"Exception caught while invoking DM: {e.ToString()}");
}
```

* the code above does the following things
    * Receives the "Alert" message passed to use from Stream Analytics on the Edge
    * parses the JSON message into an Alert object (technically, ASA passes us a JSON array of only one element, so we take the 'first' element)
    * pulls out the device ID of the device that caused the alert, and figures out, based on the 'tempstate' (HIGH or LOW) whether to send the "ON" or "OFF" command to the device
    * using the ServiceClient object, makes the Direct Method call.

Save and close the Program.cs file with CTRL-O, \<enter>, CTRL-X

## build and "publish" our module

When we build the docker image in a few moments, our project will actually be "built" as part of that process, but we'll manually go ahead and build it here just to be sure it builds without error before we try to build the docker image.

To build our code, from within the ~/edge/alertmodule folder, run

```bash
dotnet build
```

Once the code builds successfully, we are ready to build our docker image

## build our docker image

To build the docker image, run the following code, after replacing \<your docker username> with, well, your docker user name created previously

```bash
cd ~/edge/alertmodule

docker build -t "<your docker user name>/alertmodule:latest" .
```

## push our docker image

This step is optional, since we developed the image on the same box we are going to run it on, but it's good practice as you'll likely develop on a different box than you'll run Edge on in the real world.

To push your image to the Docker Hub, run

```bash
docker login -u <your docker user name> -p <your docker password>
docker push <your docker user name>/alertmodule:latest
```

## Deploy Edge module

In this section, we will get the module created above deployed and view the results.

* in the Azure portal, navigate to your IoT Hub, click on IoT Edge Devices (preview) on the left nav, click on your IoT Edge device
* click on "Set Modules" in the top menu bar.  
* In the Set Modules blade, click on "Add IoT Edge Module"
    * In the "IoT Edge Modules" dialog, give your module a name (for example:  alertmodule).  Remember the name you used, including the proper 'case' of the letters, as we'll need that when we set the routes in the next step.
    * in the image URI box, put in the exact same image name you used in the previous step (e.g. \<docker hub id>/alertmodule:latest)
    * leave the other defaults and click "Save"
* back on the "Set Modules" blade, click "next"
* on the "specify routes" blade, replace the default with the following:

```json
{
  "routes": {
    "toFormatterFromDevices": "FROM /messages/* WHERE NOT IS_DEFINED($connectionModuleId) INTO BrokeredEndpoint(\"/modules/formattermodule/inputs/input1\")",
    "toASAJob": "FROM /messages/modules/formattermodule/outputs/output1 INTO BrokeredEndpoint(\"/modules/edgeASAJob/inputs/inputFromHub\")",
    "toAlerts": "FROM /messages/modules/edgeASAJob/outputs/asaAlertTemp INTO BrokeredEndpoint(\"/modules/alertmodule/inputs/input1\")",
    "asaToIoTHub": "FROM /messages/modules/edgeASAJob/outputs/asaAggregatedTemp INTO $upstream",
    "asaAlertsToIoTHub": "FROM /messages/modules/edgeASAJob/outputs/asaAlertTemp INTO $upstream"
  }
}
```
* replace "alertmodule" above (in two places) with the name of your module, case-sensitive, that you used above if it was different than 'alertmodule'

    * the first route above, takes any message that does not come from a "module" and routes it into the input of our Formatter Module.  In other words, it takes messages that comes from downstream IoT devices and routes them into our formatter module.  It also keeps messages that comes from our formatter module from being routed back into itself.
    * the second route takes the output from our Formatter Module and routes it up to IoT Hub in the cloud

* Click "next" and then "finish" in the Azure portal

### Test our module

After a few seconds, the module should be downloaded and deployed to our IoT Edge runtime.  You can confirm this on the edge device by running "docker ps".  You should see all of the previous modules running, the edgeAgent, edgeHub, the formatterModule, the ASA module and our new AlertModule.  You can view the logs of any of them by running "docker logs -f \<module name>" to make sure they are working.

As in the previous module, start the .NET Core app that represents our IoT Device to get messages flowing into the system. 

Once that is running, you can use "docker logs -f alertmodule" (or whatever you named your module) to see its logs.  You won't see anything yet.

Watch the inputs from the python script/.NET Core app.  Once the temperture crosses above 80, you should see the Alert come through the docker logs for the alertModule, and the DM call to turn the alert "ON" come through in the debug output of the python script/.NET Core IoT Device. As the temperature drops back below 80, you should see the process repeat to turn the alert back off.

if it's not still running, run the iothub-explorer monitor-events command, as in modules 3 and 4, see the alert messages flow through to IoT Hub as well if you repeat the process.

Congratulations - You are done!  You now have:

* data coming from a IoT device flowing into IoT Edge
* an Edge module that intercepts and reformats that data
* Streaming Analytics on the edge that pre-aggregates the data as well as generates "local" alerts when a dangerous high temperature situation occurs
* a module that takes the 'high temp' alerts and uses them to invoke an action on the IoT Device in question.
