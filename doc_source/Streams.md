# Capturing Table Activity with DynamoDB Streams<a name="Streams"></a>

Many applications can benefit from the ability to capture changes to items stored in a DynamoDB table, at the point in time when such changes occur\. Here are some example use cases:
+ An application in one AWS region modifies the data in a DynamoDB table\. A second application in another AWS region reads these data modifications and writes the data to another table, creating a replica that stays in sync with the original table\.
+ A popular mobile app modifies data in a DynamoDB table, at the rate of thousands of updates per second\. Another application captures and stores data about these updates, providing near real time usage metrics for the mobile app\.
+ A global multi\-player game has a multi\-master topology, storing data in multiple AWS regions\. Each master stays in sync by consuming and replaying the changes that occur in the remote regions\.
+ An application automatically sends notifications to the mobile devices of all friends in a group as soon as one friend uploads a new picture\.
+ A new customer adds data to a DynamoDB table\. This event invokes another application that sends a welcome email to the new customer\.

DynamoDB Streams enables solutions such as these, and many others\. DynamoDB Streams captures a time\-ordered sequence of item\-level modifications in any DynamoDB table, and stores this information in a log for up to 24 hours\. Applications can access this log and view the data items as they appeared before and after they were modified, in near real time\.

A *DynamoDB stream* is an ordered flow of information about changes to items in an Amazon DynamoDB table\. When you enable a stream on a table, DynamoDB captures information about every modification to data items in the table\.

Whenever an application creates, updates, or deletes items in the table, DynamoDB Streams writes a stream record with the primary key attribute\(s\) of the items that were modified\. A *stream record* contains information about a data modification to a single item in a DynamoDB table\. You can configure the stream so that the stream records capture additional information, such as the "before" and "after" images of modified items\.

DynamoDB Streams guarantees the following:
+ Each stream record appears exactly once in the stream\.
+ For each item that is modified in a DynamoDB table, the stream records appear in the same sequence as the actual modifications to the item\.

DynamoDB Streams writes stream records in near real time, so that you can build applications that consume these streams and take action based on the contents\.

## Endpoints for DynamoDB Streams<a name="Streams.Endpoints"></a>

![\[Image NOT FOUND\]](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/images/streams-endpoints.png)![\[Image NOT FOUND\]](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/)![\[Image NOT FOUND\]](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/)

AWS maintains separate endpoints for DynamoDB and DynamoDB Streams\. To work with database tables and indexes, your application will need to access a DynamoDB endpoint\. To read and process DynamoDB Streams records, your application will need to access a DynamoDB Streams endpoint in the same region\.

The naming convention for DynamoDB Streams endpoints is `streams.dynamodb.<region>.amazonaws.com`\. For example, if you use the endpoint `dynamodb.us-west-2.amazonaws.com` to access DynamoDB, you would use the endpoint `streams.dynamodb.us-west-2.amazonaws.com` to access DynamoDB Streams\.

**Note**  
For a complete list of DynamoDB and DynamoDB Streams regions and endpoints, see [Regions and Endpoints](http://docs.aws.amazon.com/general/latest/gr/rande.html) in the AWS General Reference\.

The AWS SDKs provide separate clients for DynamoDB and DynamoDB Streams\. Depending on your requirements, your application can access a DynamoDB endpoint, a DynamoDB Streams endpoint, or both at the same time\. To connect to both endpoints, your application will need to instantiate two clients \- one for DynamoDB, and one for DynamoDB Streams\.

## Enabling a Stream<a name="Streams.Enabling"></a>

You can enable a stream on a new table when you create it\. You can also enable or disable a stream on an existing table, or change the settings of a stream\. DynamoDB Streams operates asynchronously, so there is no performance impact on a table if you enable a stream\.

The easiest way to manage DynamoDB Streams is by using the AWS Management Console\.

1. Open the DynamoDB console at [https://console\.aws\.amazon\.com/dynamodb/](https://console.aws.amazon.com/dynamodb/)\.

1. From the DynamoDB console dashboard, choose **Tables**

1. On the **Overview** tab, choose **Manage Stream**\.

1. In the **Manage Stream** window, choose the information that will be written to the stream whenever data in the table is modified:
   + **Keys only**—only the key attributes of the modified item\.
   + **New image**—the entire item, as it appears after it was modified\.
   + **Old image**—the entire item, as it appeared before it was modified\.
   + **New and old images**—both the new and the old images of the item\.

   When the settings are as you want them, choose **Enable**\.

1. \(Optional\) To disable an existing stream, choose **Manage Stream** and then choose **Disable**\.

You can also use the `CreateTable` or `UpdateTable` APIs to enable or modify a stream\. The `StreamSpecification` parameter determines how the stream is configured:
+ `StreamEnabled`—specifies whether a stream is enabled \(`true`\) or disabled \(`false`\) for the table\.
+ `StreamViewType`—specifies the information that will be written to the stream whenever data in the table is modified:
  + `KEYS_ONLY`—only the key attributes of the modified item\.
  + `NEW_IMAGE`—the entire item, as it appears after it was modified\.
  + `OLD_IMAGE`—the entire item, as it appeared before it was modified\.
  + `NEW_AND_OLD_IMAGES`—both the new and the old images of the item\.

You can enable or disable a stream at any time\. However, note that you will receive a *ResourceInUseException* if you attempt to enable a stream on a table that already has a stream, and you will receive a *ValidationException* if you attempt to disable a stream on a table which does not have a stream\.

When you set `StreamEnabled` to `true`, DynamoDB creates a new stream with a unique stream descriptor assigned to it\. If you disable and then re\-enable a stream on the table, a new stream will be created with a different stream descriptor\.

Every stream is uniquely identified by an Amazon Resource Name \(ARN\)\. Here is an example ARN for a stream on a DynamoDB table named *TestTable*:

```
arn:aws:dynamodb:us-west-2:111122223333:table/TestTable/stream/2015-05-11T21:21:33.291
```

To determine the latest stream descriptor for a table, issue a DynamoDB `DescribeTable` request and look for the `LatestStreamArn` element in the response\.

## Reading and Processing a Stream<a name="Streams.Processing"></a>

To read and process a stream, your application will need to connect to a DynamoDB Streams endpoint and issue API requests\.

A stream consists of *stream records*\. Each stream record represents a single data modification in the DynamoDB table to which the stream belongs\. Each stream record is assigned a sequence number, reflecting the order in which the record was published to the stream\.

Stream records are organized into groups, or *shards*\. Each shard acts as a container for multiple stream records, and contains information required for accessing and iterating through these records\. The stream records within a shard are removed automatically after 24 hours\.

Shards are ephemeral: They are created and deleted automatically, as needed\. Any shard can also split into multiple new shards; this also occurs automatically\. \(Note that it is also possible for a parent shard to have just one child shard\.\) A shard might split in response to high levels of write activity on its parent table, so that applications can process records from multiple shards in parallel\.

If you disable a stream, any shards that are open will be closed\.

Because shards have a lineage \(parent and children\), an application must always process a parent shard before it processes a child shard\. This will ensure that the stream records are also processed in the correct order\. \(If you use the DynamoDB Streams Kinesis Adapter, this is handled for you: Your application will process the shards and stream records in the correct order, and automatically handle new or expired shards, as well as shards that split while the application is running\. For more information, see [Using the DynamoDB Streams Kinesis Adapter to Process Stream Records](Streams.KCLAdapter.md)\.\)

The following diagram shows the relationship between a stream, shards in the stream, and stream records in the shards\.

![\[Image NOT FOUND\]](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/images/streams-terminology.png)![\[Image NOT FOUND\]](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/)![\[Image NOT FOUND\]](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/)

**Note**  
If you perform a `PutItem` or `UpdateItem` operation that does not change any data in an item, then DynamoDB Streams will *not* write a stream record for that operation\.

To access a stream and process the stream records within, you must do the following:
+ Determine the unique Amazon Resource Name \(ARN\) of the stream that you want to access\.
+ Determine which shard\(s\) in the stream contain the stream records that you are interested in\.
+ Access the shard\(s\) and retrieve the stream records that you want\.

**Note**  
No more than 2 processes at most should be reading from the same Streams shard at the same time\. Having more than 2 readers per shard may result in throttling\.

The DynamoDB Streams API provides the following actions for use by application programs:
+  `[ListStreams](http://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_streams_ListStreams.html)`—returns a list of stream descriptors for the current account and endpoint\. You can optionally request just the stream descriptors for a particular table name\.
+ `[DescribeStream](http://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_streams_DescribeStream.html)`—returns detailed information about a given stream\. The output includes a list of shards associated with the stream, including the shard IDs\.
+ `[GetShardIterator](http://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_streams_GetShardIterator.html)`—returns a *shard iterator*, which describes a location within a shard\. You can request that the iterator provide access to the oldest point, the newest point, or a particular point in the stream\.
+ `[GetRecords](http://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_streams_GetRecords.html)`—returns the stream records from within a given shard\. You must provide the shard iterator returned from a `GetShardIterator` request\.

For complete descriptions of these API actions, including example requests and responses, go to the [Amazon DynamoDB Streams API Reference](http://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_Operations_Amazon_DynamoDB_Streams.html)\.

### Data Retention Limit for DynamoDB Streams<a name="Streams.DataRetention"></a>

All data in DynamoDB Streams is subject to a 24 hour lifetime\. You can retrieve and analyze the last 24 hours of activity for any given table; however, data older than 24 hours is susceptible to trimming \(removal\) at any moment\.

If you disable a stream on a table, the data in the stream will continue to be readable for 24 hours\. After this time, the data expires and the stream records are automatically deleted\. Note that there is no mechanism for manually deleting an existing stream; you just need to wait until the retention limit expires \(24 hours\), and all the stream records will be deleted\.