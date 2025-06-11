# IoT Operations Demo

This sample demonstrates how to use a VM running in Azure to deploy an instance of IoT Operations.  Normally this would be running on a physical device on premise, but for the purposes of this demo we will use a virtual machine.

Note, this sample does not take into account device security when connecting, so do not open it to the public internet!!

## IoT Operations Setup

1. Create a Ubuntu 24.04 virtual machine with a size of Standard D4S v3.
1. Create a Public IP address for the VM.

### Arc VM Setup

We need to disable the ability for the VM to talk to the Azure Instance Metadata Service (IMDS) so that it can be used as a device in IoT Operations.  This is done by disabling the firewall and reconfiguring the hostname.

1. SSH into the VM using the public IP address.
2. Disable the firewall
```bash
sudo ufw --force enable
sudo ufw deny out from any to 169.254.169.254
sudo ufw default allow incoming
sudo apt-get update
```
3. Reconfigure the hostname
```bash
sudo -E /bin/sh -c 'echo $VMNAME > /etc/hostname'
```
4. Install the Arc Agent
```bash
wget https://aka.ms/azcmagent -O ~/install_linux_azcmagent.sh
sudo bash ~/install_linux_azcmagent.sh
sudo azcmagent connect --subscription-id "Production" --resource-group "HybridServers" --location "eastus2" #Modify the subscription-id, resource-group, and location as needed
```

### Install K3S

1. Follow the instructions at https://learn.microsoft.com/azure/iot-operations/deploy-iot-ops/howto-prepare-cluster?tabs=ubuntu

### Install IoT Operations

1. Follow the instructions at https://learn.microsoft.com/en-us/azure/iot-operations/deploy-iot-ops/howto-deploy-iot-operations

### Enable Access to the MQTT Broker

1. Open your IoT Operations resource in the Azure portal.
2. Navigate to the "MQTT Broker" tab.
3. In the MQTT broker listener for LoadBalancer, click 'Create'.
4. Give it a Name, Service Type of LoadBalancer.
5. Click +Add port entry 
    - Port 1883
    - Authentication None
    - Authorization None 
    - Protocol MQTT
6. Click 'Save'.
7. Navigate to the NSG for the IoT Operations Azure VM.
8. Add an inbound security rule for your IP address to allow traffic on port 1883.

## Connect a Device to IoT Operations

1. Create a physical device or use a tool like MQTTX and connect to either the public IP address of the VM or the DNS name of the IoT Operations resource.
1. Send data to the topic of your choice.

## Monitor the Telemetry

1. Connect to the MQTT Broker using a tool like MQTTX.
2. Subscribe to the topic you sent data to.

## Send Data to Azure

1. Create a Data Flow Endpoint for Event Hubs at https://learn.microsoft.com/en-us/azure/iot-operations/connect-to-cloud/howto-configure-kafka-endpoint?tabs=portal
1. Create a Data Flow using the Default Message Broker as the source and the Event Hubs endpoint as the destination.  Create any transformations you wish.  Follow the instructions at https://learn.microsoft.com/en-us/azure/iot-operations/connect-to-cloud/howto-configure-kafka-endpoint?tabs=portal

## Process the Data in Fabric Real Time Intelligence

1. Create a Fabric Capacity.
2. Create a Fabric Workspace.
3. Create an Event House.
4. Create an Event Stream that uses the Event Hub as the source, and the Event House as the destination.

