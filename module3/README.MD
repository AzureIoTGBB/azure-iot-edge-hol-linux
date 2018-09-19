# Azure IoT Edge Hands On Labs - Module 3

Created and maintained by the Microsoft Azure IoT Global Black Belts

## Introduction

For this step of the lab, we are going to create an IoT Edge module that take the raw CSV format provided by our IoT Device, converts it to JSON, and then puts the messages back on the edge Hub for further processing.

We will develop our module in C# using .NET Core.  In this version of the lab, rather than having a development machine set up (and having to go through the pre-reqs for that), we will do the work entirely on our Linux VM running our Edge box.

## install the IoT Edge project template

the IoT Edge team has provided a .NET Core project template that makes development of our module much easier.  To install it, run the following command.

```bash
dotnet new -i Microsoft.Azure.IoT.Edge.Module
```

## generate the project 'skeleton'

Now that our template is installed, we can create the project "skeleton" for our project.  Do do so, assuming you created a edge folder earlier, run:

```bash
cd ~/edge
dotnet new aziotedgemodule -n formattermodule
```

This will create a folder under edge called 'formattermodule'.  A number of nice things were generated for us, including a skeleton implementation of an IoT Edge Module.  So see this sample implementation, open Program.cs in your favorite Linux editor  (if you don't have a favorite, use nano)

```bash
cd formattermodule
nano Program.cs
```

Review the boiler-plate code to understand what it does.  A few notes:

* The main funtion retrieves the 'connection string' for the module from an environment variable passed into it's docker container by the edge Agent when it creates the container.  This connection string is for the module to connect to the local edge Hub instance.  Main called "Init" to initiate the connection to the local Hub and set up any callbacks.
* the Init method creates an instance of the Module Client class and connects using the connection string passed in from the edgeAgent (under the covers).  The Init method then sets any callbacks that will be raised whenever a message is routed to this module (on a named "input") or when module-twin changes happen
* The PipeMessage function is an example implementation of a "listener" for messages to be routed to this module.  In the sample implementation, we are basically getting the content of the message and any user-defined meta-data properties, and echoing them back on a new message.  We are then marking the current message as "complete" so the edge runtime knows we are done with it.  We will be modifying this sample method to implement our message 'formatter'

## Modify the sample implementation

Now lets modify the sample code to implement our Formatter module.  

* in Program.cs, in the "using" section above the Program class, add the two following 'using' statements

```CSharp
using Newtonsoft.Json;
```

* above the "Program" class, add the C# class that will represent our message we want to publish to the Hub

```CSharp
    class Telemetry
    {
        public string deviceID;
        public float temperature;
        public float humidity;
    }
```

* the balance of our work will be in the PipeMessage function.  the top part of the function, which gets the "Device Client" instance that is stored in Context and makes sure it's valid, and that opens the message and gets it's content as a string, is boiler-plate and is fine for our purposes.  The work we will be doing is inside the "if" block below (which checks to make sure we don't have an empty message):

```CSharp
    if (!string.IsNullOrEmpty(messageString))
    {
    }
```

* replace the code within the if block above, with the below code  (CTRL-K removes entire lines of code in 'nano'..  you can copy from this page and right click in putty to 'paste'..  make sure your cursor is in the right place first)

```CSharp
                // if our message isn't 11 characters long (xx.xx,yy.yy) or a comma in the 6th position
                // then it's not a message we are interested in.
                if((!(messageString.Length == 11)) || !(messageString.Substring(5,1) == ","))
                {
                    Console.WriteLine("Not a message that interests this module.");
                    return MessageResponse.Completed;
                }

                // split the CSV message into its two parts
                // humidity is first, temp second
                string[] parts = messageString.Split(",");

                string devId = "DeviceIdMissing";

                // retrieve the device ID of the sending leaf device and insert into message
                if(!string.IsNullOrEmpty(message.ConnectionDeviceId))
                        devId = message.ConnectionDeviceId;

                // create and populate our new message content
                Telemetry t = new Telemetry();
                t.deviceID = message.ConnectionDeviceId;
                t.humidity = float.Parse(parts[0]);
                t.temperature = float.Parse(parts[1]);

                // serialize to a string
                string newMessage = JsonConvert.SerializeObject(t);

                // create a new IoT Message object and copy
                // any properties from the original message
                var pipeMessage = new Message(Encoding.ASCII.GetBytes(newMessage));
                foreach (var prop in message.Properties)
                {
                    pipeMessage.Properties.Add(prop.Key, prop.Value);
                }

                // send the data to the edge Hub on a named output (for routing)
                await moduleClient.SendEventAsync("output1", pipeMessage);
                Console.WriteLine($"Converted message sent({counter}): {newMessage}");
```

* the code above does the following things
  * some rudimentary checking to make sure we have a message in the right format
  * disassembles the humidity and temp values from the CSV input
  * creates a new message using these values and serializes as JSON
  * copies any user-defined metadata properties from the original input message to the new message
  * send the message back out to the hub on named output
  * marks the original message as "complete" so that the Hub knows we successfully processed it and it does not need to be re-sent

To save the Program.cs file, hit CTRL-O and then CTRL-X to exit.

## build our module

When we build the docker image in a few moments, our project will actually be "built" as part of that process, but we'll manually go ahead and build it here just to be sure it builds without error before we try to build the docker image.

To build our code, from within the ~/edge/formattermodule folder, run

```bash
dotnet build
```

Once the code builds successfully, we are ready to build our docker image

## build our docker image

One of the nice things about our IoT Edge template is that it pregenerates the necessary "Dockerfile" to build our docker image.  That makes it simple for us to build.   It you like, you can review the Dockerfile by running

```bash
cat ~/edge/formattermodule/Dockerfile.amd64
```

Review the file and get a feel for what it's doing (which is essentially, starting from a base ".NET Core" image, copying over your code, building it, and setting it to run when the image is launched)

For the rest of the instructions "[your docker repo]" is a placeholder where you will use your docker container repository created earlier (i.e. myazurerepo.azurecr.io).

To build the docker image, run the following code.

```bash
cd ~/edge/formattermodule
docker build -f Dockerfile.amd64 -t "[your docker repo]>/formattermodule:0.0.1-amd64" .
```

## push our docker image

This step is optional, since we developed the image on the same box we are going to run it on, but it's good practice as you'll likely develop on a different box than you'll run Edge on in the real world.

To push your image to the Docker Hub, run

```bash
docker login -u [your docker user name] -p [your docker password] [your docker repo]
docker push [your docker repo]/formattermodule:0.0.1-amd64
```

where [your docker user name] is the short name of your azure container repo created earlier (i.e. myazurerepo) and [your docker password] is the container repo password you captured to Notepad in module 1  (the big long cryptographic string)

## Deploy Edge module

In this section, we will get the module created above deployed and view the results.

* in the Azure portal, navigate to your IoT Hub, click on IoT Edge Devices (preview) on the left nav, click on your IoT Edge device
* click on "Set Modules" in the top menu bar.  
* In the Set Modules blade,
  * We need to give the edgeAgent the right credentials to access our repo in order to pull the docker image for our module at run-time.  In the "Container Registry Settings" section at the top, fill in the following fields
    * Name:   [your docker user name]    (i.e. the short name of your azure container registry repo)
    * Address:  [your docker repo]   (i.e. the long name)
    * User Name:   same as "Name" above
    * Password:  the cryptographic password of your azure container repo
  * In the Deployment Modules section, click on "Add", and then "IoT Edge Module"
  * In the "IoT Edge Modules" dialog, give your module a name (for example:  formattermodule).  Remember the name you used, including the proper 'case' of the letters, as we'll need that when we set the routes in the next step.
  * in the image URI box, put in the exact same image name you used in the previous step (e.g. [your docker repo]/formattermodule:0.0.1-amd64)
  * leave the other defaults and click "Save"
* back on the "Set Modules" blade, click "next"
* on the "specify routes" blade, replace the default with the following:

```json
{
    "routes": {
        "toFormatterFromDevices": "FROM /messages/* WHERE NOT IS_DEFINED($connectionModuleId) INTO BrokeredEndpoint(\"/modules/formattermodule/inputs/input1\")",
        "toIoTHub": "FROM /messages/modules/formattermodule/outputs/output1 INTO $upstream"
    }
}
```

* replace "formattermodule" above (in two places) with the name of your module, case-sensitive, that you used above if it was different than 'formattermodule'

  * the first route above, takes any message that does not come from a "module" and routes it into the input of our Formatter Module.  In other words, it takes messages that comes from downstream IoT devices and routes them into our formatter module.  It also keeps messages that comes from our formatter module from being routed back into itself.
  * the second route takes the output from our Formatter Module and routes it up to IoT Hub in the cloud

* Click "next" and then "finish" in the Azure portal

### Test our module

After a few seconds, the module should be downloaded and deployed to our IoT Edge runtime on the Edge device.  You can confirm this by switching back to your Putty session and typing "docker ps".  You should see three modules running, the edgeAgent, edgeHub, and our new formatterModule.  You can view the logs of any of them by running "docker logs -f [module name]" to make sure they are working.

At this point, it can be useful to open a second Putty connection to our IoT Edge device.  That way you can run the .NET Core "IoT Device" in one, and look at logs, etc in the other.

As in the previous module, start the python script (or .NET Core app) that represents our IoT Device to get messages flowing into our formatter module.

Once that is running, you can use "docker logs -f formattermodule" (or whatever you named your module) to see its logs.  You should see debug output indicating that it is receiving the CSV input, and outputing our new JSON module.

### Monitor our IoT Hub

In the Azure cloud CLI that is open in the browser, if your monitoring of IoT Hub is not still running from module 2, run this command to restart the monitoring

```bash
az iot hub monitor-events -n [hub name]
```

where [hub name] is the short name of your IoT Hub.

You should see messages flowing though the hub.  These messages have come from the device, to the local Edge Hub and been forwarded to the cloud based IoT Hub in a store-and-forward fashion (i.e. transparent gateway).  Please note that there may be some delay before you see data in the monitor.

> NOTE: Now that we have an edge module "in between" our IoT device and IoT Hub in the cloud, the messages no longer appears as though it comes from the original IoT Device, but rather from our Edge device now. This is an important distinction. This is one of the reasons also, that our Formatter Module snags the original deviceID and embeds it in our message (we could have also put it in the user-defined metadata for the message). Otherwise, the original device origin would be lost.

hit CTRL-C to stop monitoring.

## Summary

Ok - now we have messages flowing from our IoT Device and being translated to JSON and sent to IoT Hub. However, the messages are flowing at the original high rate of our "dumb" IoTHub device. Now we are going to add the capability to aggregate the data at the edge, as well as determine when a "high temperature" alert condition occurs (module 4) and take local action (module 5).

To continue with Module 4, click [here](/module4)
