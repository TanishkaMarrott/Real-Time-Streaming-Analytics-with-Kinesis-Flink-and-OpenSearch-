# Real-Time Streaming Analytics with Kinesis, Flink and OpenSearch

This Real-time Streaming pipeline integrates Data Processing with Data Ingestion, Transformation, Storage, Analysis, and Visualization, creating a robust end-to-end solution.  
Services leveraged include  Kinesis, Lambda, Glue, OpenSearch. 

There's been a strong emphasis on Design Considerations that align with the overarching Architectural Design,  
I've prioritized scalability, fault tolerance, security, and performance optimization across all system layers.

## Index

- [Project Workflow](#project-workflow)
- [The Data Ingestion Layer - Specifics](#the-data-ingestion-layer---specifics)
  - [Service Utilised](#service-utilised)
  - [Primary Objective](#primary-objective)
  - [Key Design Considerations I've Made](#key-design-considerations-ive-made)
- [The Producer Codebase - In a Gist](#the-producer-codebase---in-a-gist)
- [Strategy I've Leveraged for Effective Thread Management](#strategy-ive-leveraged-for-effective-thread-management)
- [The Data Transformation Layer](#the-data-transformation-layer)
  - [Why Firehose + Glue?](#why-firehose--glue)
  - [Data Enrichment through Lambda](#data-enrichment-through-lambda)
  - [Design Considerations in this layer](#design-considerations-in-this-layer)
- [Stream Processing & Visualisation](#stream-processing--visualisation)
  - [Services](#service-utilised)
  - [The Workflow](#the-workflow)
  - [The Flink Application Codebase](#the-flink-application-codebase)
- [Wrap-Up](#wrap-up)
- [Acknowledgements](#acknowledgements)


## Project Workflow 

![Real-Time Streaming Analytics with Kinesis (1)](https://github.com/TanishkaMarrott/Real-Time-Streaming-Analytics-with-Kinesis-Flink-and-OpenSearch/assets/78227704/99edb176-7c2f-485d-bae0-b32629942201)



## The Data Ingestion Layer - Specifics

### _Service Utilised:-_
Kinesis Data Streams

### _Objective:-_
Capturing & ingesting real-time streaming data ---> serving as a pivotal bridge between data producers and consumers.


## Key Design Considerations I've Made

### _The Data Injection Mechanism_

  I've leveraged **_Kinesis Producer Library_** for constructing our Data Producers.  
  
  My Quick Breakdown:-
  
| **What it does?** | **What does it mean?** |
|-------------|-----------------|
| Aggregates records | Combines multiple records ---> enhance throughput and reduce overhead. |
| Asynchronous requests | Separates data production from stream interaction ---> uninterrupted data flow. |
| Error handling | Strategically retries failures to boost system resilience. |
| Data compression | Minimizes Data transmission size to save costs and bandwidth. |
| Metrics collection | Gathers data on volume and latency. This helps identify performance bottlenecks. |

### _Which Capacity Mode did I opt for?_

I've opted for the **_On-demand Capacity Mode_** for Kinesis Data Streams due to the _unpredictable and variable nature_ of my data stream's throughput requirements. 

With this mode, the capacity of the data stream **_scales automatically_** based on the incoming data volume, ensuring that I **don't need to predefine or manage shard capacities**. This flexibility is crucial for **_accommodating sudden spikes_** in data ingestion rates or **adjusting to changing application demands**.
  </br>

## The Producer Codebase - In a Gist

In our project, the **Kinesis Producer** is developed in Java, tailored to handle **NYC Taxi Telemetry Data**. 
The code begins by parsing data from a CSV file into Trip Objects, then configures the Kinesis Producer with parameters like **Record Buffer Time** and **Maximum Connections**.

We've incorporated **parallelism and concurrency** by deploying an **`ExecutorService`** with a fixed thread pool size, optimizing scalability and throughput. The integration of **`CompletableFuture`** with `ExecutorService` facilitates a fully **non-blocking asynchronous processing**, enhancing the efficiency of task management across threads.

And finally, **graceful Error handling** is achieved through the output of shard IDs for success and logging of failures, ensuring a **robust Data Ingestion Layer**.
</br>

## Strategy I've leveraged for Effective Thread Management

### **_The Pain-Point:-_** 

Submitting a task to the _'`ExecutorService`'_ is asynchronous. However, upon submitting the task, it returns a `Future` object immediately, which helps in tracking the status and retrieving the result at a later point.
_**`Future.get()` forces the calling thread to wait. this makes the solution only partially asynchronous. Not recommended**_

### **_Our Solution:-_**

_`ExecutorService`_ + _`CompletableFuture`_

I've used a combination of both, since `CompletableFuture` provides non-blocking methods like `thenApply()` and `thenCombine()` to handle asynchronous task results. These execute post the completion of Future. My entire workflow is now fully asynchronous. It allows chaining and managing tasks efficiently without blocking the calling thread.

</br>

## The Data Transformation Layer

Here, I'd be using _**Kinesis Data Firehose**_, in conjuction with _**AWS Glue**_.

### _Why did I use Firehose + Glue?_

&#8594; We'd be using **_KDF for capturing and loading streaming data_** reliably into Data Stores / Analytical Tools (In our case, S3 would be our Data Store).
It's fully managed, and scales automatically to match the throughput of incoming data. However, it can help with only _minimal _processing

**_Rationale behind using Glue_**  
As a central Metadata Repository through Data Catalog. The Schema Definitions it stores, enhances querying capabilities in Athena. **Athena can use the Schema Information from the Data Catalog for querying data stored in S3.**
(I've shared the Table Definition above, Firehose references this definition in Glue)

## Lambda? For enriching the data...

&#8594; Designed to processes streaming data, focusing on data transformation and standardisation. Sets up logging for monitoring, **converts pickupDate and dropoffDate fields to ISO 8601 format.** Having decoded the records from base-64, it **inserts the source 'NYCTAXI' column.**
Function has been designed to handle errors, generating responses for each processed record, and manages batch processing as well.

</br>

## Design Considerations in this layer

### _**Conversion of Source record format:-**_

 In the scope of our project, **Kinesis Data Firehose** has been leveraged for both data delivery into S3 and preliminary data transformation.
 **conversion from JSON to Parquet format**. 
 
 Couple of Reasons here- **a) Significantly reduces Storage Costs**. **b) Parquet's columnar structure** allows for more efficient data querying in Athena.
 
  </br>
  
### _Optimising the Buffer Size and Interval-_

&#8594; I've opted to **_maximize the Buffer Interval time_** for data delivery into **Amazon S3**. 

**Rationale behind this:-** By allowing Data to accumulate in large batches before delivery, we're **reducing the number of PUT requests to S3**, thereby reducing transaction costs. This also results in **improvising the throughput** through batching and subsequent storage. Something around **300-600 seconds** would be a good number to start with.

&#8594; Buffer Size has been maximised, Costs would be lowered, but at the cost of a **higher latency**. 

&#8594; Cranking up the Buffer Interval to **900 seconds** (max possible) would be a relative choice. ***We need to strike balance between the **timely availability of data versus the operational costs** incurred.****

  </br>
  
### _Compression and Encryption for S3 -_

&#8594; I've utilized **Snappy compression** for source records, which leads to faster transmission and cost savings in storage. I'm prioritising **high speed over a higher compression ratio**.
 
&#8594; **Encryption** is implemented through **AWS-owned keys** for security and confidentiality of data as it moves through the Firehose stream, particularly crucial when converting data formats like JSON to Parquet.


## **Stream Processing & Visualisation**

### **_Services_** 
Kinesis Data Analytics (KDA)

### **_The Workflow_** 
This is **Workflow #2** , As we've mentioned, Data is ingested through KDS in the form of JSON Blobs. 

The streaming data is then processed using a **Flink Application** deployed on  **Kinesis Data Analytics**. Flink excels at **extracting real-time insights** from Streaming Data. So when its **Huge Volumes + Huge Velocity**, Flink goes beyond traditional SQL.
It's also useful for some **complex eventful processing**, windowing, and **stateful computations** / operations.

OpenSearch is a really powerful **Visualiser**, it's designed to work on **Streaming data**, and the high level of **scalability** that comes with it. It's used for **searching, storing and analysing** Streaming data, Log Data. It's a Search and Analytics Engine, synonymous to **Historical Data Analysis**, and Visualisation.

## _The Flink Application Codebase_

-  We've defined the Kinesis Connector, that enables the Flink App to read code from the Stream, and the OpenSearch Connector that enables writing processed storage in OpenSearch Connector, for storage and analysis
  
- Created **`_taxi_trips_`** table, which is linked to the Kinesis stream. This is not a real table --- This is virtually created in the Flink Ebvironment, It maps to the structure of KDS, facilitating its processing
  
- **The `trip_statistics` table in OpenSearch.** This table would basically set up to store aggregated data in OS, like trip counts, and averageduration
  
- A **_series of analytical queries to extract insights_** such as average trip duration, distance, peak travel times, and frequent locations.
  
- Have performed Future Analysis and Visualisation through an **_aggregation query to insert summarized data_** into the `trip_statistics` table.

## Wrapping it Up

_So, finally yes! We're through!_

Our pipeline does present an end-to-end solution for real-time data processing and analysis. It encompasses data ingestion, processing, storage, and visualization:

**_Data Ingestion with Kinesis Data Streams:_** Efficiently captures streaming data.

**_Processing and Analysis:_** With Kinesis Data Firehose for preliminary transformations and loading into S3.
Using Flink in Kinesis Data Analytics for real-time, complex data processing.

**_Data Storage and Visualization:_** S3 for durable storage of processed data.
OpenSearch for data querying, analysis, and visualization.



## Acknowledgements

Really appreciate AWS Workshops for their resources and support in this project's development.
 [AWS Workshops](https://www.aws.training/learningobject/curriculum?id=20685).








    

