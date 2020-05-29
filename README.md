# Demo of Azure Data Explorer Batching and Streaming Ingestion

This page includes 
* docuemnt link to create Azure Data Explorer and Event Hubs
* an Event Hubs client in C# to send event to Event Hub, it sends out 1 event per second
* Kusto script to create Table and JSON Mapping
* Kusto script to enable streaming ingestion of the table

**It shows the max event delay is 5 minutes in batching ingestion, after turn on streaming ingestion, the delay less than 1 seconds of all the events.**

### 1. Create Azure Data Explorer and create table with the script below

https://docs.microsoft.com/en-us/azure/data-explorer/create-cluster-database-portal

```
.create table TestTable (TimeStamp: datetime, Name: string, Metric: int, Source: string, EventHubEnqueuedTime: datetime, EventHubOffset: string) 
.create table TestTable ingestion json mapping 'TestMapping' '[{"column":"TimeStamp", "Properties": {"Path": "$.timeStamp"}},{"column":"Name", "Properties": {"Path":"$.name"}} ,{"column":"Metric", "Properties": {"Path":"$.metric"}}, {"column":"Source", "Properties": {"Path":"$.source"}},{ "column" : "EventHubEnqueuedTime", "Properties":{"Path":"$.x-opt-enqueued-time"}},{ "column" : "EventHubOffset", "Properties":{"Path":"$.x-opt-offset"}}]'
.alter table TestTable policy ingestiontime true
```

### 2. Create EventHub and et ingestion connection in Azure Data Explorer
Follow this document to setup the demo environment

https://docs.microsoft.com/en-us/azure/data-explorer/ingest-data-event-hub

Select x-opt-enqueued-time and x-opt-offset in event system properties

### 3. Build Azure EventHub client
Clone the code from 

https://github.com/Azure-Samples/event-hubs-dotnet-ingest

and replace the code with below and input the eventhub name
```csharp
using Microsoft.ServiceBus.Messaging;
using System;
using System.Collections.Generic;
using System.Text;
using System.Threading;
using System.Linq;

namespace SendSampleData
{
    class Program
    {
        const string eventHubName = "Your eventhub name";
        // Copy the connection string ("Connection string-primary key") from your Event Hub namespace.
        const string connectionString = @"<Your eventhub connection string>";

        static void Main(string[] args)
        {
            EventHubIngestion();
        }

        static void EventHubIngestion()
        {
            var eventHubClient = EventHubClient.CreateFromConnectionString(connectionString, eventHubName);
            int counter = 0;
            for (int i = 0; i < 1000; i++)
            {
                int recordsPerMessage = 1;
                try
                {
                    List<string> records = Enumerable
                        .Range(0, recordsPerMessage)
                        .Select(recordNumber => $"{{\"timeStamp\": \"{DateTime.UtcNow}\", \"name\": \"{$"name {counter}"}\", \"metric\": {counter + recordNumber}, \"source\": \"EventHubMessage\"}}")
                        .ToList();
                    string recordString = string.Join(Environment.NewLine, records);
                    Console.WriteLine($"message content {recordString}");
                    EventData eventData = new EventData(Encoding.UTF8.GetBytes(recordString));
                    Console.WriteLine($"sending message {counter}");
                    // Optional "dynamic routing" properties for the database, table, and mapping you created. 
                    // eventData.Properties.Add("Table", "TestTable");
                    // eventData.Properties.Add("IngestionMappingReference", "TestMapping");
                    // eventData.Properties.Add("Format", "json");
                    eventHubClient.Send(eventData);
                }
                catch (Exception exception)
                {
                    Console.ForegroundColor = ConsoleColor.Red;
                    Console.WriteLine("{0} > Exception: {1}", DateTime.Now, exception.Message);
                    Console.ResetColor();
                }

                counter += recordsPerMessage;

                Thread.Sleep(1000);
            }
        }
    }
}

```

### 4. Check the batching ingestion
Open the Azure Explorer Web UI

https://dataexplorer.azure.com/

And run the query 

```
TestTable
 | extend ingestionTime = ingestion_time()
```

From the result, the max delayed records look like as below 

```
"TimeStamp": 2020-05-28T06:15:13Z,
"Name": name 425,
"Metric": 425,
"Source": EventHubMessage,
"EventHubEnqueuedTime": ,
"EventHubOffset": ,
"ingestionTime": 2020-05-28T06:20:14.3726331Z,
```

**ingestionTime - TimeStamp = delay time**

TimeStamp is the eventtime, ingestionTime is the Azure Data Explorer ingrestion time. 

**The latency is around 5 minutes.**

This behavior is consistent with the description of the document

> The IngestionBatching policy can be set on databases, or tables. By default, if not policy is defined, Kusto will use a default value of **5 minutes** as the maximum delay time, 1000 items, total size of 1G for batching.

https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/batchingpolicy#details
 
### 5. Enable the streaming ingestion

Enable streaming ingestion in cluster with the document

https://docs.microsoft.com/en-us/azure/data-explorer/ingest-data-streaming#enable-streaming-ingestion-on-your-cluster

Enable streaming ingestion in the table with kusto script

```
.alter table TestTable policy streamingingestion '{  "NumberOfRowStores": 4}'
```

### 6. Check the streaming ingestion

Open the Azure Explorer Web UI

https://dataexplorer.azure.com/

And run the query 

```
TestTable
 | extend ingestionTime = ingestion_time()
```

And after enabling the streaming ingestion, the records look like as below: 

```
"TimeStamp": 2020-05-29T08:56:19Z,
"Name": name 123,
"Metric": 123,
"Source": EventHubMessage,
"EventHubEnqueuedTime": 2020-05-29T08:56:20.876Z,
"EventHubOffset": 8886544,
"ingestionTime": 2020-05-29T08:56:20.9163495Z,
```

**The latency is around 1 second.**
