# Azure IoT Edge Hands On Labs - Module 1

Created and maintained by the Microsoft Azure IoT Global Black Belts

## Create an IoT Hub and an "Edge Device"

For the lab exercises, we need an IoT Hub created in an Azure Subscription for which you have administrative access.

Create an IoT Hub in your subscription by following the instructions [here](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-create-through-portal).   NOTE:  you can stop after the section titled "Create the IoT Hub"  (i.e. BEFORE "Change the settings of the IoT Hub").  There is no need to go any further in those instructions after the IoT Hub is created.

While you are in the Azure portal, let's go ahead and grab a couple of important connection parameters and create an IoT Edge Device

In the IoT Hub blade of the Azure portal for your created IoT Hub, do the following:

* In the left-hand nav bar, click on "Shared Access Policies" and then click on "iothubowner", copy the "Connection String - Primary Key" string and paste it into Notepad.  We'll need it later.  This is your "IoTHub Owner Connection string".  (keep that in mind, or make a note next to it in Notepad, we will use it in subsequent labs).
* Close the "Shared Access Policy" blade
* In the left-hand nav bar, click on "IoT Edge Devices (preview)"
* click "Add Edge Device"
* Give your IoT Edge Device a name and click "Create"
* once created, find the IoT Edge Device connection string (primary key) and copy/paste this into Notepad.  This is the "IoT Edge Device" connection string

## Create docker hub account

IoT Edge modules are pulled by the Edge runtime from a docker containder image repository.  You can host one locally in your own network/infrastrcuture if you choose, Azure offers a [container service](https://azure.microsoft.com/en-us/services/container-service/)  and of course, Docker themselves offer a respository (docker hub).  For simplicity, we will run the labs based off of hosting images in docker hub.  If you feel confident in doing so, feel free to leverage other docker image responsitories instead of docker hub if you wish.

For Docker Hub, you need a Docker ID.  Create one by visting www.docker.com and clicking on "Create Docker ID" and following the instructions.  If you are given a choice during sign up, choose a repository visibility of 'public'.  Remember the docker ID you create, as we'll use it later.  Generally, docker images are referred to in a three part name:  \<respository>/image:tag where "respository", if using Docker Hub, is just your Docker ID,  image is your image name, and tag is an optional "tag" you can use to have multiple images with the same name (often used for versioning).

## Set up Ubuntu VM in Azure

* open http://portal.azure.com in your browser and log in with your azure credentials (which may have been provided to you, if you are attending a live session)
* in the upper left corner of the portal, click "Create a resource"
* In the search bar, search for "Ubuntu Server"
* select the most recent version of Ubuntu Server"  (which as of this writing is "Ubuntu Server 17.10")
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

Go get a cup of coffee.  Deployment will take approximately 5 minutes.

## Connect to your VM

To connect to our VM, we need a terminal program.  If you already have one, feel free to use it.  If you don't, we recommend Putty.

* You can download Putty [here](https://the.earth.li/~sgtatham/putty/latest/w32/putty.exe).  Just download and save to your desktop.

We will connect to our new Linux box via it's public IP address.  To get the connection details:

* in the Azure portal, if it didn't automatically take you there after provisision, then navigate to your new VM
* on the Overview tab, click on "Connect"
* note the connection details of your VM, which will be in the form \<username>@x.x.x.x where x.x.x.x is the public IP of your VM
* launch Putty and ensure the Connection Type is the default of SSH.  In "Host Name (or IP Address)", enter the details from the previous step, and click "Open"
* you will likely get a "Putty Security Alert", if so, click "yes"
* enter the password you chose when you created the VM above to complete your connection

## Install Linux VM Prerequisites

In order to execute the hands-on labs, there are a number of pre-requisites that need to be installed and configured.  Unless otherwise noted, the default installation of the items below are fine

### Update the package list

We want to make sure the install package list on our VM is up-to-date with the latest, so run

```bash
sudo apt-get update
```

### install pip

The 'control' script for the Azure IoT Edge is based in python, and is supplied as a pip package.  To install it, need pip, which isn't installed by default on our ubuntu image.

to install pip, run

```bash
sudo apt-get install python-pip
```

### install Docker for Linux

The IoT Edge modules are run and managed via Docker containers.  Ubuntu doesn't come with Docker preinstalled, so we need to install it.  

To install docker, run the following commands:

```bash
curl -fsSL get.docker.com -o get-docker.sh

sudo sh get-docker.sh
```

Once done, we want to add our account to the Docker admin group (so we don't have to use 'sudo' every time to interact with docker)

NOTE:  this it the linux login that you used to access your VM, NOT the docker ID you just created.

Run:
```bash
sudo usermod -aG docker <your user name>
```

Once the command has complete, you'll need to log out of your session ('exit') and re-connect as you did previously with Putty

### Install the Azure IoT Edge control script

The Azure IoT Edge control script gives us local control over configurating and running the Edge runtime.  To install the Azure IoT Edge control script.  Run the following command:

```bash
sudo pip install -U azure-iot-edge-runtime-ctl
```

### Install the iothub-explorer tool

We will be using the iothub-explorer tool to spy on the messages being sent to IoT Hub in the cloud.  To install it, we first need to install Node.JS and NPM

```bash
sudo apt-get install nodejs

sudo apt-get install npm
```

One that is done, we can install the iothub-explorer tool and test to make sure it's working ok by just dumping out the version

```bash
sudo npm install -g iothub-explorer

iothub-explorer --version
```

the last command above should dump out a version 1.x.x where x may vary.

## Additional miscellaneous setup

There are a few final steps needed to set up our specific lab scenario.  We are using our Edge device "as a gateway*, so we need a) our IoT Device to be able to find it and b) to have valid certificates so the IoT Device will open a successful TLS connection to the Edge

* clone the "preview" branch of the Azure IoT C SDK (we need it to get to some scripts for generating certificates)

```bash
git clone -b CACertToolEdge https://github.com/Azure/azure-iot-sdk-c
```

* Add a host file entry for our Edge device -- this will let our "IoT Device" resolve and find our Edge gateway.  To do this:
    * run 'sudo nano /etc/hosts'
        * add a row at the bottom with the following
        * 127.0.0.1  mygateway.local
    * save and close the file  (CTRL-O, CTRL-X)
    * confirm you can successfully "ping mygateway.local"

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

* we need to register our cert with IoT Hub.  To do so:

    * you will need to get the cert from the linux VM to the local box.  You can ftp to the VM to pull it off it you need to.  The file you need is ./certs/azure-iot-test-only.root.ca.cert.pem
    * in the azure portal, navigate back to your IoT Hub and click on "Certificates" on the left-nav and click "+Add". Give your certificate a name, and upload the pem file
* we are now ready to create our Edge device-specific certs.  To do so, run:

```bash
./certGen.sh create_edge_device_certificate myGateway
```

This creates the public key, private key, etc for the device.  Now we need to create the public key full chain, by running the following command from the 'certs' directory:

```bash
cd certs

cat ./new-edge-device.cert.pem ./azure-iot-test-only.intermediate.cert.pem ./azure-iot-test-only.root.ca.cert.pem > ./new-edge-device-full-chain.cert.pem
```

## install certs

In order for our end IoT Device to connect to the gateway, it needs to trust the certs we just generated.  To do so, we need to install that cert.  To install it, 

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

## Configure and start IoT Edge

Now that we have all the pieces in place, we are ready to start up our IoT Edge device.  We will start it by specifying the IoT Edge Device connection string capture above, as well as specifying the certificates we generated to allow downstream devices to establish valid TLS sessions with our Edge gateway.

To setup and configure our IoT Edge device, run the following command  (if you used '1234' for the password above, enter it again here when prompted).

```bash 
cd ~

sudo iotedgectl setup --connection-string "<Iot Edge Device connection string>" --edge-hostname mygateway.local --device-ca-cert-file /home/<your user id>/edge/certs/new-edge-device.cert.pem --device-ca-chain-cert-file /home/<your user id>/edge/certs/new-edge-device-full-chain.cert.pem --device-ca-private-key-file /home/<your user id>/edge/private/new-edge-device.key.pem --owner-ca-cert-file /home/<your user id>/edge/certs/azure-iot-test-only.root.ca.cert.pem
```

Replace *IoT Edge Device connection string* with the Edge device connection string you captured above.  Also, replace \<your user id> with your linux login user id.  (hint: it's probably easier if you copy/paste from here into notepad first and you can search and replace with CTRL-H).  When you run the command, if it prompts you for a password for the edge private cert, use '12345'

We're ready now to start our IoT Edge device

```bash
sudo iotedgectl start
```

You can see the status of the docker images by running 

```bash
docker ps
```

at this point (because we haven't added any modules to our Edge device yet), you should only see one container/module running called 'edgeAgent'

If you want to see if the edge Agent successfully started, run

```bash
docker logs -f edgeAgent
```

Note that you may see an error in the edgeAgent logs about having an 'empty configuration'.  That's fine, because we haven't set a configuration yet!  :-)

CTRL-C to exit the logs when you are ready

__**Congratulations -- you now have an IoT Edge device up and running and ready to use**__

To continue with Module 2, click [here](/module2)
