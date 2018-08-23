# Azure IoT Edge Hands On Labs - Module 1

Created and maintained by the Microsoft Azure IoT Global Black Belts

## Set up Ubuntu VM in Azure

As mentioned, for this lab instance, we are going to leverage a VM running in Azure as our Edge device.  It will take a few minutes to create the VM, so we'll start that process and do some other prep work while it builds.  To get started, 

* open http://portal.azure.com in your browser and log in with your azure credentials (which may have been provided to you, if you are attending a live session)
* in the upper left corner of the portal, click "Create a resource"
* In the search bar, search for "Ubuntu Server"
* select the most recent version of Ubuntu Server"  (which as of this writing is "Ubuntu Server 18.04")
* click "Create"
* in the Basics blade
    * give your VM a name
    * choose a username you can remember
    * for Authentication type, change the selection to "Password"
    * enter a strong password that you can remember and confirm it
    * for "Resource group", choose the default of "Create new" and enter a unique name for your Resource Group
    * for "Location", pick a location near you  (if you don't care, the default of Central US is fine)
    * click "ok"
* In the "Size" blade, choose a size for your VM.  Any size will work, but we would recommend at least 4GB of RAM (8GB would be better)
* In the "Settings" blade, leave all the defaults and hit "Ok"
* On the summary blade, review the details and legalese and, if you agree, click "Create"

While the VM builds, let's get started with our IoT Hub and devices.

## Create an IoT Hub, "Edge Device", and "Leaf Device"

For the lab exercises, we need an IoT Hub created in an Azure Subscription for which you have administrative access.

### Create the IoT hub

While you can use the GUI to create an IoT Hub, for expediency, we are going to use the Azure Cloud Shell and the Azure Command Line Interface built into the Azure Portal.  

* In the open Azur portal, click on the CLI button in the tool bar (highlighted below)

![azure cli](/images/cloud-shell.png)

* once the shell opens, run the commands under the "Create an IoT Hub" found [here](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-create-using-cli#create-an-iot-hub)
* once the IoT Hub has been succesfully created, run the following command to grab the connection string for the IoT Hub.  You will need it later.

``` bash
az iot hub show-connection-string --name [your iothub name]
```
* open Notepad and copy/paste the connection string value (without the quotes).  We will call this (for later use) the "IoTHub Owner Connection String"

### Create the Edge device

* Using the cloud shell command line opened previously, install the azure iot cli extension using the following command

``` bash
az extension add --name azure-cli-iot-ext
```

* Execute the following command to create and register an "Edge Device" with the IoT Hub to represent our Edge device

``` bash
az iot hub device-identity create --device-id [device id] --hub-name [hub name] --edge-enabled
```

* for [device id], pick a descriptive name for your Edge device.  For [hub name], use the IoT Hub name created above

* once the device is created, run the following command to retrieve the connection string for the device:

``` bash
az iot hub device-identity show-connection-string --device-id [device id] --hub-name [hub name]
```

* copy/paste the returned connection string (minux the quotes) into Notepad for use later.  Label this one "Edge Device Connection String"

### Create "leaf" device

* Using the cloud shell command line opened previously, execute the following command to create and register a "Leaf" device with the IoT Hub.  The "Leaf" device will represent our downstream device that talks to our sensor and sends IoT data through the edge device

Note that we removed the --edge-enabled parameter from the end of the command, as this is a regular IoT device, not an Edge device

``` bash
az iot hub device-identity create --device-id [device id] --hub-name [hub name]
```

* for [device id], pick a descriptive name for your Leaf device.  For [hub name], use the IoT Hub name created above

* once the device is created, run the following command to retrieve the connection string for the device:

``` bash
az iot hub device-identity show-connection-string --device-id [device id] --hub-name [hub name]
```

* copy/paste the returned connection string (minux the quotes) into Notepad for use later.  Label this one "Leaf Device Connection String"

## Create Azure Container Registry

IoT Edge modules are pulled by the Edge runtime from a docker compatible container image repository.  You can host one locally in your own network/infrastrcuture if you choose, Azure offers a [container service](https://azure.microsoft.com/en-us/services/container-service/)  and of course, Docker themselves offer a respository (docker hub).  

For the labs, we will run the labs based off of hosting images in an Azure Container Registry.  If you feel confident in doing so, feel free to leverage other docker image repositories instead of azure if you wish.

We will continue to use the Azure cloud shell CLI to create our container registry.  

* To get started, execute the following command in the CLI:

```bash
az acr create --resource-group [resouce group] --name [container registry name] --sku Basic
```
* for [resource group], use the same resource group name created earlier.  For [container registry name], create a unique name for your container registry.  The name will be used in the URI of our registry (in the form of [registryname].azurecr.io), so use a URI friendly name, preferably lower case and no spaces

the next step, we will need to gather our container registry credentials.  To gather the credentials, execute the following commands

```
az acr update -n [container registry name] --admin-enabled true
az acr credential show --name [container registry name]
```

* you will see the primary and secondary passwords for your regisry.  Copy/paste your registry password into notepad for use later

## Connect to your VM

In the Azure portal, navigate to the "Virtual Machines" blade and see if our VM has finished provisioning (it should show the "Running" state).  If not, grab some coffee!

To connect to our VM, we need a terminal program.  If you already have one, feel free to use it.  If you don't, we recommend Putty.

* You can download Putty [here](https://the.earth.li/~sgtatham/putty/latest/w32/putty.exe).  Just download and save to your desktop.

We will connect to our new Linux box via it's public IP address.  To get the connection details:

* * on the Overview tab of your VM, click on "Connect"
* note the connection details of your VM, which will be in the form \<username>@x.x.x.x where x.x.x.x is the public IP of your VM
* launch Putty and ensure the Connection Type is the default of SSH.  In "Host Name (or IP Address)", enter the details from the previous step, and click "Open"
* you will likely get a "Putty Security Alert", if so, click "yes"
* enter the password you chose when you created the VM above to complete your connection

## Install prereqs and tools

### install .NET Core

# TODO:  update these instructions...  I think they are broken

To develop our Edge modules later, we are going to do it in .NET Core.  This is primary because the tooling is further along, a this preview stage of IoT Edge, for .NET Core/C# than our other SDK languages.  So first, we need to install .NET Core on our Linux box in order to compile our code

The process to install .NET Core on Linux in general is outlined [here](https://docs.microsoft.com/en-us/dotnet/core/linux-prerequisites?tabs=netcore2x).

However, since we are running Ubuntu 17.10, we've distilled the instructions down for you.  Run the following commands on our Ubuntu box

```bash
curl https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > microsoft.gpg
sudo mv microsoft.gpg /etc/apt/trusted.gpg.d/microsoft.gpg

sudo sh -c 'echo "deb [arch=amd64] https://packages.microsoft.com/repos/microsoft-ubuntu-artful-prod artful main" > /etc/apt/sources.list.d/dotnetdev.list'
sudo apt-get update

sudo apt-get install dotnet-sdk-2.1.3
```

Once the instructions above are complete, check for a successful install by running

```bash
dotnet --version
```

it should report back version 2.1.3

## Install IoT Edge

Now that we've done the preliminary work, it's time to install IoT Edge on our device.  We will first install IoT Edge in "quick start" mode, and then add the necessary certificates to allow our "leaf" device to connect.

To install IoT Edge in quick start mode, follow the instructions [here](https://docs.microsoft.com/en-us/azure/iot-edge/how-to-install-iot-edge-linux)

(the connection string you use in the config.yaml file is your "Edge Device Connection String copied earlier)

## Additional miscellaneous setup

There are a few final steps needed to set up our specific lab scenario.  We are using our Edge device *as a gateway*, so we need a) our IoT Device to be able to find it and b) to have valid certificates so the IoT Device will open a successful TLS connection to the Edge

* Add a host file entry for our Edge device -- this will let our "IoT Device" resolve and find our Edge gateway.  To do this:
    * run 'sudo nano /etc/hosts'
        * add a row at the bottom with the following
        * 127.0.0.1  mygateway.local
    * save and close the file  (CTRL-O, CTRL-X)
    * confirm you can successfully "ping mygateway.local"

* clone the Azure IoT C SDK (we need it to get to some "convenience scripts for generating our certificates).  From your "home" directory on your VM, run

```bash
git clone https://github.com/Azure/azure-iot-sdk-c
```

When a 'downstream' device connects to IoT Hub, it will do so over a TLS connection.  In order for that secure connection to happen, we need our Edge gateway to have certificates that can be trusted by the IoT device.

To generate these certificates:

from the bash command prompt
* make an 'edge' folder   (mkdir edge)
* cd to the edge folder (cd edge)
* we need to copy the cert scripts here, so run these commands

```bash
cp ~/azure-iot-sdk-c/tools/CACertificates/*.cnf .
cp ~/azure-iot-sdk-c/tools/CACertificates/*.sh .
```

* make the certGen.sh script executable

```bash
chmod 700 certGen.sh
```

* next we need to create the certificate chain (root and intermediate certs)

```bash
./certGen.sh create_root_and_intermediate
```

* we are now ready to create our Edge device-specific certs.  To do so, run:

```bash
./certGen.sh create_edge_device_certificate mygatewayca
```

This creates the public key, private key, etc for the device.  Now we need to create the public key full chain, by running the following command:

```bash
cat ./certs/new-edge-device.cert.pem ./certs/azure-iot-test-only.intermediate.cert.pem ./certs/azure-iot-test-only.root.ca.cert.pem > ./certs/new-edge-device-full-chain.cert.pem
```

## install certs

In order for our end IoT Device to connect to the gateway, it needs to trust the certs we just generated.  To do so, we need to install that cert.  To install it,

# TODO:  can we switch to the PG's instructions??

* run the following commands

```bash
cd ~/edge/certs

openssl x509 -in azure-iot-test-only.root.ca.cert.pem -inform PEM -out azure-iot-test-only.root.ca.cert.crt

sudo mkdir /usr/share/ca-certificates/extra

sudo cp azure-iot-test-only.root.ca.cert.crt /usr/share/ca-certificates/extra/azure-iot-test-only.root.ca.cert.crt

sudo dpkg-reconfigure ca-certificates
```

* with the last command, you will get a screen warning about installing certs to the root store.  Ensure that "yes" is highlighted and hit \<enter> to pick "Ok"
* on the next screen, you'll be presented a list of certs that will be installed.  Make sure there is a "*" next to the azure iot cert from above (there probably is not, so hit the \<space bar> to toggle one on), then hit \<tab> to move to the "Ok" button, and hit \<enter>

if you want to double check to see if the certs were installed, run

```bash
sudo cat /etc/ca-certificates.conf
```

at the bottom of the file, you should see "extra/azure-iot-test-only.root.ca.cert.crt"

## Update IoT Edge configuration

Now we will update the IoT Edge configuration to use our new certificates.

The first step is to edit the config.yaml file to tell Edge to use our certificates vs. the "quick start" certs that were automatically created when Edge was installed...

* Edit the config.yaml file

```
sudo nano /etc/iotedge/config.yaml
```

* scroll down to the "Certificate Settings" section, uncomment the 'certificates:' line and the three lines below it  (note that in a config.yaml file, spacing is important..  'certificates' should be left justified, and the lines below should be EXACTLY two spaces indented)

* update the certificate paths to the following values (replacing \<your linux user id> with the user name with which you logged into our linux VM:

```bash
certificates:
  device_ca_cert: "/home/<your linux user id>/edge/certs/new-edge-device-full-chain.cert.pem"
  device_ca_pk: "/home/<your linux user id>/edge/private/new-edge-device.key.pem"
  trusted_ca_certs: "/home/<your linux user id>/edge/certs/azure-iot-test-only.root.ca.cert.pem"
```

* scroll down to the "Edge device hostname" section and edit the hostname to be 'mygateway.local', like this..

```bash
hostname: "mygateway.local"
```

* hit CTRL-O, \<enter>, to save and then CTRL-X to exit.  

* Ensure that the Linux service manager picks up the changes to our config file by running:

```bash
sudo systemctl daemon-reload
```

There is a temporary workaround needed because of a bug in IoT Edge.  We need to remove the "quick start" certificates to ensure the new certificates get used

``` bash
sudo rm -rf /var/lib/iotedge/hsm/certs
sudo rm -rf /var/lib/iotedge/hsm/cert_keys
```

* restart IoT Edge

```
sudo systemctl restart iotedge
```

## Confirm IoT Edge status

Confirm that the IoT Edge daemon started by running

```bash
sudo systemctl status iotedge
```

the status should be "active (running)".   CTRL-C to exit

You can see the status of the docker images by running 

```bash
sudo docker ps
```

at this point (because we haven't added any modules to our Edge device yet), you should only see one container/module running called 'edgeAgent'

If you want to see if the edge Agent successfully started, run

```bash
sudo docker logs -f edgeAgent
```

Note that you may see an error in the edgeAgent logs about having an 'empty configuration'.  That's fine, because we haven't set a configuration yet!  :-)

CTRL-C to exit the logs when you are ready

__**Congratulations -- you now have an IoT Edge device up and running and ready to use**__

To continue with Module 2, click [here](/module2)
