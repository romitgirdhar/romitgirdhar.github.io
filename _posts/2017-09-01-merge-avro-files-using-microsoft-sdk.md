---
layout: post
title: "Merge Avro Files Generated from Azure EventHubs Capture using Microsoft.Avro SDK"
comments: true
date: 2017-09-01
---

Recently, while working on an IoT scenario, we decided to use Event Hubs Capture to store events received by an Azure Event Hubs into Azure Blob Storage. There were 2 Azure Event Hubs setup (1 for the sensor readings and the other for the reference data). The requirement was to use the reference data sent to Azure Event Hubs to process the sensor data using Azure Stream Analytics in near real-time.

**Challenge**: Event Hubs Capture supports a maximum of 15-minute windows for archival (thus having a minimum for 4 files per hour). However, Stream Analytics only accepts 1 reference data file per hour. 

**Proposed Solution**: Use a time-based triggered Azure function that runs once an hour, merges the archived Avro files into one file, which will then be used by Stream Analytics as reference data. Additionally, one of the features of Event Hubs Capture is the ability to store events in Avro format. Avro is a data serialization system and is commonly used for big data workloads. Given that fact that Stream Analytics also supports the Avro format, we decided to merge multiple Avro files into one.

**Architecture**:

**Merging Avro Files Together**
I setup a quick C# console application to merge Avro files together on an hourly basis. This was later turned into an Azure function. But, to keep this simple, we’re going use the C# Console application.
The Event Hubs capture path is setup as follows: <EventHubsNamespace>/<EventHubsName>/<Year>/<Month>/<Day>/<Hour>/<EventHubsPartition>
To accommodate this path, the first thing to do in our application is to setup the path as follows:
