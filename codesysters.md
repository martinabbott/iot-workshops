# IoT 101 Workshop

The idea behind this workshop is to give a general overview of the sort of processes and components involved in a basic IoT solution. IoT solutions can be thought of as a value chain that goes from Things at one end to Action at the other.

![IoT Value Chain](/images/iotvaluechain.png)

The end solution makes use of a device simulator to represent a Thing that sends telemetry messages to Azure IoT Hub, an Azure Stream Analytics job takes the telemetry and sends to Azure Cosmos DB, and if time permits the creation of a simple report using Power BI Desktop to represent Action to visualize the telemetry.

![Solution Architecture](/images/architecture.png)

For the workshop you'll use Microsoft Learn that provides a sandbox subscription within Microsoft Azure, use the Azure Command Line Interface and the Azure Portal.

To complete the workshop you need to download and install Power BI Desktop. The application is free to use and can be installed from the Microsoft Store on a Windows 10 laptop. 

The workshop has the following tasks:

1. Sign in to Microsoft Learn and start a sandbox (3 minutes)
1. Create an Azure IoT Hub (3 minutes)
1. Register a device with IoT Hub (2 minutes)
1. Create an Azure Cosmos DB account (10 minutes)
1. Create an Azure Stream Analytics job (10 minutes)
1. Set up the Raspberry Pi Simulator (2 minutes)
1. Install Power BI Desktop (5 minutes)
1. Create a Power BI report (15 minutes)

Approximate total time required: 50 minutes

## Sign in to Microsoft Learn and start a sandbox

To use Microsoft Learn you need to use a Microsoft Account, such as name@outlook.com or name@hotmail.com.

If you do not have a Microsoft Account, you can sign up for one at [Outlook.com](https://outlook.live.com/owa/).

Once you have a Microsoft Account, go to [Microsoft Learn](https://docs.microsoft.com/en-au/learn/) and sign in.

There are many great modules in Microsoft Learn and it is worth returning once you complete this workshop to take a look around at other modules you can do  that are of interest. 

For now, go the [Analyze images in real-time with machine learning](https://docs.microsoft.com/en-au/learn/modules/build-ml-model-with-azure-stream-analytics/) module.

This module requires you to use an Azure Sandbox so is a good point to kick off from. In order to activate the sandbox, scroll down to the list of tasks and go to task 3, **Create storage account to hold photos**.

Once loaded, the module task will check to see if you have activated a sandbox in your current session. If you have not, click the **Activate sandbox** button to get started.

## Create an Azure IoT Hub
You will use the Azure Command Line Interface (CLI) to create an Azure IoT Hub that will act as the ingestion point for telemetry from your device simulator.

To do this, you need to open the [Azure Portal](https://portal.azure.com/) and sign in with the same email address you used for Microsoft Learn.

If you get a message telling you that you do not have any active subscriptions, click on your name in the top right hand corner of the browser window and select **Switch directory** and choose **Microsoft Learn Sandbox**.

Once you are in the right directory, click the Cloud Shell button to the right of the search bar at the top of the portal. 

![Cloud Shell](/images/cloudshell.png)

You are going to assign the IoT Hub name to an environment variable named HUB_NAME for convenience. This name needs to be globally unique so choose something that is  combination of name and additional random numbers and letter. Replace **{hub name}** in the following command with the name chosen for your IoT Hub and execute the command in the Cloud Shell.

```shell
export HUB_NAME={hub name}
```

In order to create your IoT Hub, you need to know the name of the resource group that has been created in the sandbox environment. To find out, execute the following command in the Cloud Shell.

```shell
az group list
```

Highlight and copy the text between the quotes starting with **Learn-**.

Execute the following command in the Cloud Shell to create an IoT Hub in the resource group, replacing **[sandox resource group name]** with the text previously copied. 

```shell
az iot hub create --name $HUB_NAME --resource-group [sandbox resource group name] --location southcentralus --sku F1
```

The --sku F1 parameter configures the IoT hub to use the free F1 pricing tier, which supports up to 8,000 events per day. However, Azure subscriptions are limited to one free IoT hub each. If the command fails because you've already created a free IoT hub, specify --sku S1 instead. The S1 tier greatly expands the message-per-day limit, but isn't free.

Verify the IoT Hub has been created correctly by executing the following command in the Cloud Shell.

```shell
az iot hub list
```

## Register a device with IoT hub
In order to add a device to the newly created IoT Hub you need to install the IoT Hub CLI extension.

Install this by executing the following command.

```shell
az extension add --name azure-cli-iot-ext
```

The following commands first list the devices that are already provisioned within the created IoT Hub, which should be an empty list, and then creates a device called ***mypi*** in the IoT Hub Device Registry.

```shell
az iot hub device-identity list --hub-name $HUB_NAME
az iot hub device-identity create --hub-name $HUB_NAME -d myrpi
```

For the next part of the workshop, you'll need the connection string that IoT Hub assigned to the new device. Find this by executing the following command.

```shell
az iot hub device-identity show-connection-string --device-id myrpi --hub-name $HUB_NAME
```

Copy the text between the quotes that starts with HostName in the resulting JSON text.

```json
{
  "connectionString": "[Copy text between these quotes]"
}
```

The device that has been created can send events to the IoT Hub using one of several protocols, including HTTPS, AMQP, and MQTT.

## Set up Raspberry Pi Simulator

You will use the [Raspberry Pi Simulator](https://azure-samples.github.io/raspberry-pi-web-simulator/) to send telemetry data to IoT Hub, that you will then send to Azure Cosmos DB using an Azure Stream Analytics job.

To configure the simulator you need to provide the connection string you copied in the previous step. Go to line 15 in the code of the device simulator and replace the text (including the brackets) with the text you copied.

![Configure Simulator](./images/pi-sim.jpg)

```javascript
const connectionString = '[Your IoT hub device connection string]';
```

At this point you have finished setting up the simulator, but when you have completed the steps for Cosmos DB and Stream Analytics, you can return to the simulator and click Run to start sending telemetry data.

## Create a Cosmos DB account
Before creating the Stream Analytics job that pushes data in to our data store, we need to first create the data store.

For the purpose of this workshop you will use Azure Cosmos DB which provides a great way to store large volumes of unstructured data that is very efficient, has low latency for reading data and supports geo-distribution.

You will create the database account, a database and a collection that will store our raw telemetry.

You can use the Azure CLI again to create what you need. Let's start by creating a database account, to do this execute the following command, remembering to replace **[sandbox resource group name]** with the resource group name you copied previously and **[unique database name]** with something of your choice that should be unique such as name and some random characters..

```shell
az cosmosdb create --name [unique database name] --resource-group [sandbox resource group name] --locations regionName=southcentralus failoverPriority=0
```

Next you create a database using the following command, again remembering to replace **[sandbox resource group name]** with our resource group name.

```shell
az cosmosdb database create --name [unique database name] --resource-group [sandbox resource group name] --db-name codesystersdb
```

The final thing you need to do is to create our database collection. In order to create the collection you need to get the primary master key associated with the database account. To get that, execute the following command.

```shell
az cosmosdb keys list --name [unique database name] --resource-group [sandbox resource group name]
```

You should get output similar to the following.

```json
{
  "primaryMasterKey": "[primary master key]",
  "primaryReadonlyMasterKey": "[primary read only key]",
  "secondaryMasterKey": "[secondary master key]",
  "secondaryReadonlyMasterKey": "[secondary read only key]"
}
```

Copy the value of **[primary master key]** and the execute the following command remembering to replace the **[sandbox resource group name]**, **[unique database name]** and **[primary master key]** with the values you have copied.

```shell
az cosmosdb collection create --collection-name telemetry --resource-group [sandbox resource group name] --db-name codesystersdb --partition-key-path "/deviceId" --throughput 400 --name [unique database name] --key [primary master key]
```

You will now use the Azure Portal to check that the database account, database and collection have been created correctly.

In the Search bar at the top of the portal, type **resource groups**, and click the top link to show all resource groups within the subscription, there should be just one. Click the link to open the resource group which will start with **Learn-**.

Click on the Cosmos DB account created earlier, the click Data Explorer in the side menu.

Your database and collection should look like the following.

![Data Explorer](/images/dataexplorer.png)

This completes this task and next you start looking at Azure Stream Analytics.

## Create a Stream Analytics job

You can create the Stream Analytics job using PowerShell, but for this task you will use the Azure Portal as this will help you get familiar with how to create and configure services without the Azure CLI.

Make sure you are in the Resource Group for the sandbox, it will start with **Leave-**.

Click the **+** to add a new resource, and in the box that appears, type **Stream Analytics**.

This should present a list with **Stream Analytics job** as the first choice. Choose this and  then complete the form that appears.

You will need to give the job a name, suggest something like **passthrough** as the purpose of this job will be to simply shape the data and let it pass through to Cosmos DB.

![Stream Analytics](/images/streamanalytics.png)

Choose the resource group and make sure the Cloud is selected rather than Edge. Once complete click **Create**.

The job should only take a few seconds to create. Navigate back to the Resource Group and then to the newly created job by clicking on the job link in the list of resources.

To configure the job you need to provide an input, an output and create a query that will extract relevant information from the inbound telemetry before sending it to Azure Cosmos DB.

Clicking Inputs allows you to add a new input to the stream. In this case we are going to add an IoT Hub input stream. Once clicked a dialogue box will open allowing you to configure the input. 

Give the input a name, **iothub**, and use the option to **Select from existing subscription**. Choose the IoT Hub you created earlier.

![Stream Analytics Input](/images/asainput.png)

The process to create an output follows the same pattern. Go in to Outputs and select Add and choose Cosmos DB.

Give the output a name, **cosmosdb**, and use the option to **Select from existing subscription**. Choose the account and database created previously, and for **Collection pattern** enter **telemetry**. For **Partition Key**, enter **/deviceId**

![Stream Analytics Output](/images/asaoutput.png) 

Once the input and outputs are configured, you need to write a query to get the required data out of the inbound telemetry message stream. To do this, click **Edit query* and enter the following SQL command.

```sql
SELECT 
	deviceId,
	temperature,
	humidity,
	EVENTENQUEUEDTIME as eventtime
FROM 	iothub
INTO
	cosmosdb
```

![Stream Analytics Query](/images/asaquery.png)

The SQL used by Stream Analytics is a subset of full T-SQL, and is covered extensively in the Microsoft documentation. The language allows the usual SQL type commands such as **GROUP BY**, but also allows different types of time-bound operations such as **TUMBLINGWINDOW**. Furthermore, it is possible to perform Anomaly Detection right within the query, and user defined functions can be written to help perform calculations and functions not available directly within the language.

Once complete, click the **Run** button, keep the default of **Now** and start the job. As the input is a stream the start process allows you to either choose the current time, a custom time in the past or the last time the job was stopped. 

Next we will install Power BI Desktop and create a simple report to display our data. In order to create the report, data needs to be being received by Cosmos DB, so now go back to the Raspberry Pi Simulator and click the Run button.

## Install Power BI Desktop

Microsoft Power BI Desktop can be found in the Microsoft Store on your laptop. If you open the store and search for Power BI Desktop you should be able to install it. Once installed, click the Launch button within the store and launch from within the start menu.

## Create a Power BI report

In this final task you will create a simple report to display the temperature and humidity for our device as a function of time.

First you need to get data in to Power BI Desktop, to do this you click the **Get Data** button in the ribbon and choose **More...** at the bottom of the list. Choose **Azure** from the left column and scroll down to choose **Azure Cosmos DB (beta)** from the right column. Click **Connect** to continue.

![Power BI Desktop Get Data](/images/pbigetdata.png)

You need to enter the URL of the database. This can be found by using the Azure Portal and navigating to the Cosmos DB account. From the Overview tab, the URL is displayed in the format https://[unique database account name].documents-azure.net:443. 

Next you need to enter the Account Key which is the value of the primary master key you copied earlier in the workshop. If you need to get it from the Azure Portal, click Keys in the menu when in the Cosmos DB account.

Once set, Power BI Desktop will read the data source and bring back a list of collections and their content. Since you only have a single collection, the items that make up our record within Cosmos DB will be displayed. Deselect everything except `deviceId`, `temperature`, `humidity` and `eventtime`.

Once complete the data will be imported from Cosmos DB in to Power BI Desktop and displayed.

Initially the display will just show as Records. This is because Power BI Desktop has not yet expanded the data within the records sent by the Raspberry Pi Simulator. To expand that, click the *Expand** icon in the top right of the table.

The table should expand and now include columns for `Document.deviceId`, `Document.temperature`, `Document.humidity` and `Document.eventtime`.

Let's give the columns a more friendly name to remove the phrase `Document.` from the column names. To do this, right click in the column heading of each column and remove the text to be left with just the text after the period.

You need to change the data types of the columns since Power BI Desktop assumes all data is text initially. You can do this by right clicking the column heading again, and choosing **Data Type**. The list of available data types flies out and you need to choose the following for the temperature, humidity and eventtime columns. The deviceId column can remain as **Text** so you do not need to change that. 

Column | Data Type 
------------ | ------------
temperature | Decimal Number 
humidity | Decimal Number
eventtime | Date/Time

![Power BI Data Types](/images/pbidatatypes.png)

Once done, click the Close and Apply button in the ribbon to import the data.

Once this process has completed, go to the Modelling tab in the Power BI ribbon as you need to adjust what happens to the imported data as by default, Power BI Desktop assumes that numeric data needs to be summarized.

By clicking each column in turn, provide the following settings. Again the deviceId column can remain at its default values.

Column | Format | Default Summarization    
------------ | ------------ | ------------ 
temperature | Decimal number (5 decimal places) | Do not summarize
humidity | Decimal number (5 decimal places) | Do not summarize
eventtime | Leave as Default | Do not summarize

![Power BI Desktop Summarize](/images/pbisummarize.png)

You have now imported the initial data and modelled the data to provide a mechanism to create the report. To create the report choose the top icon on the left hand menu.

You will create 2 line graphs, one for temperature and one for humidity. 

![Power BI Desktop Visualization](/images/pbivisualize.png)

Click the Line Graph icon under Visualization and a graph outline will appear on the report canvas. Expand it a little so you get more information.

Next provide the `deviceId` as the legend, `temperature` as the Values and `eventtime` as the Axis. You will find this shows a graph with a dot in the middle. Go the Visualization information and ensure the setting for Axis shows eventtime only and **not** Date Hierarchy, and that *temperature* is set to Max rather than Count.

![Power BI Event Time Setting](/images/pbieventtime.png)

![Power BI Desktop Temperature Setting](/images/pbitemperature.png)

Repeat the process for humidity. 

Once done you can refresh the data by clicking the **Refresh** button in the ribbon. If you were to publish the report, you would be able to set it to automatically refresh based on a schedule.

However, with the desktop functionality, repeatedly clicking **Refresh** should show you that data is being received by IoT Hub, is being analyzed by Stream Analytics and then stored in Cosmos DB. 

After completing this task, you have concluded the workshop, hope you enjoyed it.


