# Integration platform assignment
**Task:**
Imaginary customer needs an integration platform in the cloud. 
All the messages are in JSON format 
and there should be an API layer that exposes the integrations outside the platform in controlled manner. 
Both synchronous and asynchronous integration patterns should be used. 
On the sake of simplicity, you don’t have to take into account the integration message size.

 

The platform should take the following into account:
- Authentication, authorization
- Scalability
- Redundancy

Also, it should be very good if you could describe the approaches setting up the environment 
and maintaining it in cases that there is need for a change to it. 
How about information security. Are there some kind of approaches that you would prefer?
___
**assumptions:**
The assumption is that the customer for whom we are building the integration platform is responsible for data producers.
(in real case, those would be verified with the client)
Therefore, it is assumed the data source is inside the cloud or AWS SDK is used to write data to appropriate service in AWS.

> **Note:** the selection of the services and the final design would also depend on the specific requirements, type of data and other factors that are unknown to me
___
The suggested solution utilizes Kinesis Data Streams service provided by AWS. 
An alternative could be SQS. However, Kinesis has few advantages over it such as possibility to have multiple consumer applications and promised better performance.

In Kinesis Data Streams, data is ordered based on the arrival in the service and expires after retention period passed. 
The default retention period (24 hours) should be sufficient - meaning each data item in the service is automatically removed after one day.

The horizontal scalability in Kinesis Data Streams is achieved with shards. Data is distributed in multiple shards and its number can be adjusted base on the amount of data. Each shard is independent and failure in one does not affect the other. Also, they are replicated over multiple availability zones achieving redundancy and high availability.
A partition key needs to be chosen well to achieve good utilisation of shards. In case of need to read data directly from Kinesis Data Streams, knowing the partition key value might be helpful to determine in which shard the data is stored (MD5 algorithm and 128-bit integer hash key is used to calculate hash key).
However, reading data directly from Kinesis Data Streams (at least with boto3 AWS SDK) is inefficient and slow (I compared the performance with other options in [research paper](ICSA2025_pre_submit.pdf)).

While data is in data streams, it can be processed by consumer applications. Consumer applications read data from shards and use shard iterators to continue in processing the data where they stopped.

I would suggest to handle data separately for synchronous and asynchronous approach of data delivery. This would provide isolation to ensure that if something goes wrong in one, the other one is not blocked.

For **synchronous** approach, I would use data stores. In case of time-series data, I would prefer specialized time-series database (i.e. Timestream) which would allow to leverage from extensive query options and built-in time-series functions. Also, tools like Grafana work well with Timestream.
Otherwise, I would use KV NoSQL database DynamoDB. A benefit is high horizontal scalability (data is distributed into partitions based on partition key) and ability to handle heterogeneous data - the structure does not need to be known (ideal for messages in JSON format).
The data in data stores would not be stored permanently. Retention period or time-to-live would be set to achieve automatic data deletion after some time so data store does not grow too big and does not generate unnecessarily high cost.

A simple endpoint could be implemented using API Gateway and lambda functions. For synchronous data delivery, consumers could call the endpoint which would query recent data from the data stores.
I would prefer using API keys for authentication (due to simplicity and easiness of their use), and distribute them individually to the parties using the integration platform.
Of course, all data transfer would be secured and use HTTPS protocol.
Alternatively, to achieve higher security, I would opt for Oauth 2.0 with Client Key and Client secret (for M2M) as described [here](https://aws.amazon.com/blogs/mt/configuring-machine-to-machine-authentication-with-amazon-cognito-and-amazon-api-gateway-part-2/).

