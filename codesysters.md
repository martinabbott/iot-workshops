# Codesysters IoT 101 Workshop

[Raspberry Pi Simulator](https://azure-samples.github.io/raspberry-pi-web-simulator/)

<a href="http://example.com/" target="_blank">example</a>

## Create an Azure IoT Hub
Define a name for the IoT hub and assign it to an environment variable named HUB_NAME. The name must be globally unique since it will be used as part of the public DNS name.
bash

```shell
export HUB_NAME={hub name}
```

Execute the following command in the Cloud Shell to create an IoT Hub in the resource group:
Azure CLI

```shell
az group list
```

```shell
az iot hub create 
    --name $HUB_NAME 
    --resource-group [sandbox resource group name] 
    --location southcentralus 
    --sku F1
```

The --sku F1 parameter configures the IoT hub to use the free F1 pricing tier, which supports up to 8,000 events per day. However, Azure subscriptions are limited to one free IoT hub each. If the command fails because you've already created a free IoT hub, specify --sku S1 instead. The S1 tier greatly expands the message-per-day limit, but isn't free.

## Register sample devices with the IoT hub

Devices that transmit events to an Azure IoT hub must be registered with that IoT hub. Once registered, a device can send events to the IoT hub using one of several protocols, including HTTPS, AMQP, and MQTT.

We'll create a Node.js app that registers an array of simulated cameras with the IoT hub you've created. The Azure Cloud Shell you're working in has Node.js installed.

Create a directory in the Cloud Shell to serve as the project directory. Then cd to that directory in a Command Prompt or terminal window.

```sql
SELECT System.Timestamp as [Time Ending],
    COUNT(*) AS [Times Triggered]
FROM CameraInput TIMESTAMP BY timestamp
GROUP BY TumblingWindow(n, 1)
```
