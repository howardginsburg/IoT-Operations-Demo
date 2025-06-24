# IoT Operations Real Time Intelligence Demo

This sample demonstrates how to use an [Azure IoT Dev Kit](https://microsoft.github.io/azure-iot-developer-kit/) to send telemetry data to Azure IoT Operations, and then to process that data in [Microsoft Fabric Real Time Intelligence](https://learn.microsoft.com/en-us/fabric/real-time-intelligence/overview). 

Normally, Azure IoT Operations would be running on a physical device on premise, but for this sample, we will use VM running in Azure.
Later we will connect an MXChip to the MQTT broker and send telemetry data to it.

Note, this sample does not take into account device security when connecting, so do not open it to the public internet!!

## Prerequisites

1. Ubuntu 24.04 virtual machine with a size of Standard D4S v3.
1. Public IP address for the VM.
1. Ability to SSH into the VM via port opening, JIT, or Bastion.
1. Open port 1883 on the VM for MQTT traffic.
1. Azure Event Hubs resource.
1. Fabric Capcity and Workspace.

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

## Connect an MXChip to the MQTT Broker

1. Follow the instructions at https://github.com/howardginsburg/MXChipMQTTSample to deploy the firmware used for this sample to your MXChip.
    - Since there is no security on the MQTT broker, you can use any name for the device, such as "office" and any password.
2. The telemetry data sent by the MXChip will be in the format:
```json
{
  "device": "office",
  "mac": "c8:93:46:88:8b:32",
  "temperature": 78.08,
  "humidity": 51.1,
  "pressure": 980.3,
  "gyroX": 560,
  "gyroY": -2170,
  "gyroZ": 350,
  "buttonA": 0,
  "buttonB": 0
}
```

## Monitor the Telemetry

1. Connect to the MQTT Broker using a tool like [MQTTX](https://mqttx.app/).
2. Subscribe to the topic you sent data to.

## Send Data to Azure

1. Create a [Data Flow Endpoint for Event Hubs](https://learn.microsoft.com/en-us/azure/iot-operations/connect-to-cloud/howto-configure-kafka-endpoint?tabs=portal).
2. Create a [Data Flow](https://learn.microsoft.com/en-us/azure/iot-operations/connect-to-cloud/howto-configure-kafka-endpoint?tabs=portal)
    - Use the Default Message Broker as the source.
    - Create a Compute transformation named 'mqtttopic' that uses the custom formula `@$metadata.topic` to add the mqtt topic used to the payload.
    - Event Hubs endpoint as the destination.

This will augment the telemetry that is sent to the Event Hub to look like:
```json
{
  "device": "office",
  "mac": "c8:93:46:88:8b:32",
  "temperature": 78.08,
  "humidity": 51.1,
  "pressure": 980.3,
  "gyroX": 560,
  "gyroY": -2170,
  "gyroZ": 350,
  "buttonA": 0,
  "buttonB": 0,
  "mqtttopic": "/devices/office/messages/events/"
}
```

## Process the Data in Fabric Real Time Intelligence

1. Create an Event House.
2. Create a raw table for the data.
```kql
.create table mxchip(buttonA:long,buttonB:long,device:string,gyroX:long,gyroY:long,gyroZ:long,humidity:dynamic,mac:string,mqtttopic:string,pressure:dynamic,temperature:dynamic,EventProcessedUtcTime:datetime,PartitionId:long,EventEnqueuedUtcTime:datetime,current_time:datetime,deviceDateTime:string)
```
3. Create an Event Stream
    - Use the Event Hub as the source.
    - Add a Transformation that creates a `current_time` field.
    - Event House as the destination using a new table called `mxchip`.
4. Save and start the Event Stream.

## Transform the data into a silver layer

1. Create a silver table.
```kql
.create table mxchip_silver (buttonA: long, buttonB: long, device: string, gyroX: long, gyroY: long, gyroZ: long, humidity: dynamic, mac: string, pressure: dynamic, temperature: dynamic, messageTime: datetime)
```
2. Create a function to transform the raw data into the silver table.  The format of deviceDateTime is not in ISO and we will fix it here.
```kql
.create-or-alter function MxChipRawData() {
    mxchip 
    | extend deviceDateTime_millis = replace(@":(\d{3})$", @".\1", deviceDateTime) 
    | extend deviceDateTime_iso = replace(@"(\d{4})/(\d{2})/(\d{2})", @"\1-\2-\3", deviceDateTime_millis)
    | extend messageTime = todatetime(deviceDateTime_iso) 
    | project buttonA, buttonB, device, gyroX, gyroY, gyroZ, humidity, mac, pressure, temperature, messageTime
}
```
3. Create an update policy to transform the raw data into the silver table.
```kql
.alter table mxchip_silver policy update 
```[{
    "IsEnabled": true,
    "Source": "mxchip",
    "Query": "MxChipRawData()",
    "IsTransactional": false,
    "PropagateIngestionProperties": false
}]```
```


## Real Time Dashboard

1. Create a new dashboard in Fabric.
2. Create a new Parameter called '_device' with a kql query of
```kql
mxchip_silver
| where messageTime  between (['_startTime'] .. ['_endTime'])
| distinct device
| project device
```kql
1. Query for the average temperature:
```kql
mxchip_silver
| where messageTime between (_startTime .. _endTime)
| where device in (_devices) or isempty(_devices)
| summarize avgTemperature=round(avg(toreal(temperature)), 2) by mac, device, quarter_hour=bin(messageTime, 15m)
| sort by mac, quarter_hour
```
2. Pin the tile to a dashboard.
3. Query for the number of messages sent by each device in the last 24 hours:
```kql
mxchip_silver
| where messageTime  between (_startTime .. _endTime)
| where device in (_devices) or isempty(_devices)
| extend quarter_hour = bin(messageTime, 15m)
| summarize records_count = count() by mac, device, quarter_hour
| project mac, device, quarter_hour, records_count
```
4. Pin the tile to a dashboard.
