# Design Patterns – IoT Data and Aggregation. 

In this design pattern, you will learn how you can insert IoT data with high throughput and then have aggregations on different fields. 

To understand this design pattern, you should already be familiar with Azure Cosmos DB and have a good understanding of change feed, request unit (RU) and Azure Functions. Please follow the link to learn about it and then come back to this article. 

Many databases achieve extreme high throughput and low latency because they partition the data. This is true for all the NoSQL products i.e. MongoDB, HBase, Cassandra or Azure Cosmos DB. All these databases can scale out unlimitedly because of partitioning or sharding.  

Let us look at Azure Cosmos DB more closely. On the top level, a container is defined. You can think of container as table, a collection, or a graph; this is the main entity, which holds all the information. Azure Cosmos DB uses the word “Container” to define this top-level entity, because Cosmos DB is multi-model database, so this container is synonymous to Collection for SQL and Graph API, and called Table for Cassandra or Table API.  

A collection has many physical partitions and the number of partitions are allocated based on the  throughput requirement  of a collection. You can have one partition or five or ten partitions depending on your throughput requirement. 

To achieve high scale throughput and low latency, you need to specify a partition key and row key while inserting the data, and use the same partition key and row key while reading the data. If you choose the right partition key then your data will be distributed evenly across all the partitions, and the read and write operations can be in single digit milliseconds. 
Internally, Azure Cosmos DB uses hash-based partitioning. When you write an item, Azure Cosmos DB hashes the partition key value and uses the hashed result to determine which partition to store the item in. A good partition key will distribute your data equally among all the available partitions as shown below. 

(../media/GoodPartition.png "Fig. 1")

Please note, there isn’t a one to one mapping between partition keys and physical partitions, which means a million partition keys will not create million physical partitions. Ten million partition keys can be stored in ten physical partitions. Often novice users think that a partition key is equal to physical partition. Please remember, one is a logical concept (Partition key) and the other is a physical concept. They are not mapped one to one. Because of hashing and modulo operators, many partition keys mapped to few partitions. Each logical partition can store 10 GB of data, when the data grows more than 10 GB, this partition is automatically split. You never have to worry about splitting partitions; Azure Cosmos DB does it behind the scene. However, you should never have a partition key, which may have more than 10 GB of data.

>Million-partition key will not create million physical partitions.

Now, let’s take a concrete example to understand it. Here is the scenario; you are working in an IoT company, which has IoT devices installed in buildings to maintain the temperature, you have hundreds and thousands of customers all around the world. Each customer is having hundreds and thousands of IoT devices and they are updating the temperature every minute. So, let us define how data will look:

```
{   
    CustomerId: Microsoft,
    DeviceId: XYZ-23443,
    Temprature: 68
    DateTime: 32423512
}
```

Imagine you have a customer which is a global company with offices in every country, have 100,000 IoT devices. These IoT devices sending 2 KB of data every minute. With this speed, one-day data can be of 2 GB. With this rate, you may fill the partition in 5 days. However, you can use TTL mechanism, to delete the data automatically, but for our discussion sake, let’s assume you have to keep this data for 30 days.

If you choose, CustomerId as the partition key, you will see your data is skewed   for large customers and your partitions   will look as shown below. 

(https://azurecosmosdb.github.io/blogs/media/BadPartition.png "Fig. 2")

This kind of partitioning will also create throttling for large customers. Large customers, who have hundreds and thousands of IoT devices and they are inserting the data in a collection partitioned on CustomerId may be throttled.  You may wonder why it will be throttled? To understand that, imagine your collection is defined to have 5000 RU/Sec and you have 5  partitions  . This means, each partition can have 1000 RU. So, it means your throughput is 1000 RU per partition. 

>Please note, we said here five partitions, but this number is again here for discussion sake. This number may change in future for your throughput. With the change in hardware, tomorrow you may just get three partitions or one for 5000 RU. Remember, these physical partitions are not constant, they will keep splitting automatically as your data will keep growing.

Users often make this mistake and then they complain that they are being throttled at 2000RU even if they have provisioned the collection for 5000 RU.  In this scenarios, the main issue is their data is not partitioned properly and they are try to insert  2000 RU data in one partition. This is the one of the important reason, that you need to have a good partition key, which can distribute your data across all the partitions.

Ok, if CustomerId is not a good partition key, then what other keys we can use? You will also not like to partition the data on DateTime, because this will create a hot partition. Imagine, if you have partitioned the data on time, then for a given minute, all the calls will be hitting one partition.  If you need to retrieve the data for a customer then it will be a fan out query because data may be distributed on all the partitions.

To choose the right partition keys, you have to think and optimize for read or write scenarios. For simpler scenarios, it is possible to get a partition key for both read and write scenarios. However, if that is not the case then you have to compromise and optimize for one of the scenario. In this article, we are exploring the scenario, where we do not have one right partition key for read and write both, but let’s see what we can do to satisfy both the read and write requirements.

In this scenario, we will optimize for writing, it is best to define the collection with DeviceId as partition key for fast ingestion. DeviceId is not only unique   but much more granular than CustomerId.  Always prefer to have a key with more cardinality, so your data will be distributed  across all the partition. However, what if for reporting you want to do aggregation over CustomerId? 

This is the crux of this pattern, that you would like to partition the data for insert scenario but for your reporting scenario, you would like to group the data on some different partition key. This is two mis-match requirement on partition key.

Imagine, you have inserted the data with DeviceId as partition key, but if you now want to group by temperature by CustomerId, your query will be a cross partition query. Cross partition queries are ok for once a while scenario  . All data is indexed in Azure Cosmos DB, so cross partition queries are not necessarily bad thing but they are expensive in nature. 

Cross-partition queries cost you much more RU then the point look up. In Cross-partition queries, you pay for latency and RU both. In cross-partition queries, every partition is searched for your data, which is much expensive operation than a point lookup in one partition.
You have two options to solve this problem  . First option is to use change feed of Cosmos DB and using Azure Function you aggregate the data per hours and then store the aggregated data in another collection, where CustomerId is the partition key.  

(https://azurecosmosdb.github.io/blogs/media/ChangefeedReporting.png "Fig. 3")

You can again listen to the change feed of Reports/Hours collection to aggregate the data for per day and store the aggregation in another Cosmos DB (Reports/day). The above picture is very self-explanatory. IoT devices are sending data directly to Cosmos DB. This pattern is possible because of change feed. Change feed exposes the append only log of Cosmos DB. The change feed includes inserts and update operations made to documents within the collection. You can read more about change feed here. However, please know that change feed is enabled by default for all account and for all collections.

To learn more about how to use change feed and azure function, check this [screen cast](https://www.youtube.com/watch?v=iprndNsUeeg).

Second option is to use Spark, to do the aggregation, and keep the aggregated value in SQL data warehouse or a second collection where partition key is CustomerId. 

(https://azurecosmosdb.github.io/blogs/media/ChangefeedSpark.png "Fig. 4")

This option will also use the change feed. From Spark, you can connect directly to change feed and get all the changes in spark at the real time. Once the data is in spark, you can do the aggregation and then write that data back to Cosmos DB or to SQL DW. 

Here is the code snippet for Spark to read the data from Cosmos DB, do the aggregation and write back the data.

```
# Base Configuration
iotConfig = {
    "Endpoint" : "https://xx.documents.azure.com:443/",
    "Masterkey" : "E0wCMaBIz==",
    "Database" : "IoT",
    "preferredRegions" : "Central US;East US2",
    "Collection" : "IoT", 
    "checkpointLocation" : "dbfs://checkpointPath"
}

# Connect via Spark connector to create Spark DataFrame
iot_df = spark.read.format("com.microsoft.azure.cosmosdb.spark").options(**iotConfig).load()
iot_df.createOrReplaceTempView("c")
psql = spark.sql ("select DeviceId, CustomerId, Temp from c")
 
writeConfig = {
    "Endpoint" : "https://xx.documents.azure.com:443/",   
    "Masterkey" : "E0wCMaBKdlALwwMhg==",
    "Database" : "IoT",
    "preferredRegions" : "Central US;East US2",
    "Collection" : "MV", 
    "Upsert" : "true"
    }
iot_df.createOrReplaceTempView("c")
psql = spark.sql ("select CustomerId, avg(temp) as Temp_Avg from c group by c.CustomerId ")
psql.write.format("com.microsoft.azure.cosmosdb.spark").mode('append').options(**writeConfig).save()

```

Check the [screen cast](https://www.youtube.com/watch?v=P9Qz4pwKm_0&t=1559s) to learn how to use Spark with Cosmos DB.

Both the options can give you per minute aggregation by listening to the live change feed.  
Depending upon your reporting requirement, you can keep different aggregations at different level in different collection or in same collection. The other option you can have to keep these aggregated values to SQL data warehouse.  

