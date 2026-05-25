Kafka Notes

Data is constantly being produced and collected. Every bite of data leads us to a next action we might take.

The less effort we spend on moving data around the more we can focus on the core business at hand.

In the pub/sub model a producer, also known as a publisher, sends a message to a centralized broker that consumers also known as subscribers can ingest and take the next action.

This is what makes Kafka a message broker. Be sending and receipt of messages are decoupled.

In contrast to a publisher client sending messages to all the different receivers. It only has to send a message to the centralized broker which reduces complexity and increases scalability in a growing system. Point to point 
Interprocess communication does not scale well.

Kafka is also described as a distributed commit log or more recently, a distributed streaming platform. A filesystem or database commit log is designed to provide a durable record of all transactions so that they can be replayed to consistently build the state of a system. Similarly, data within Kafka is stored durably, in order, and can be read deterministically. In addition, the data can be distributed within the system to provide additional protections against failures, as well as significant opportunities for scaling performance.

A unit of data in Kafka is called a message which is simply an array of bytes. A message can also have an optional piece of meta-data called a key which is also a byte array.

Keys are used when messages are to be written to partitions in a more controlled manner. The
simplest such scheme is to generate a consistent hash of the key and then select the partition number for that message by taking the result of the hash modulo the total number of partitions in the topic. This ensures that messages with the same key are always written to the same partition (provided that the partition count does not change).

For eﬀiciency, messages are written into Kafka in batches. A batch is just a collection of messages, all of which are being produced to the same topic and partition. An individual round trip across the network for each message would result in excessive overhead, and collecting messages together into a batch reduces this. Of course, this is a trade-oﬀ between latency and throughput: the larger. the batches, the more messages that can be handled per unit of time, but the longer it takes an individual message to propagate. Batches are also typically compressed, providing more eﬀicient data transfer and storage at the cost of some processing power.

To add structure to a message, a schema can be imposed. Some options for schemas include JSON, XML, and Apache Avro.  Many Kafka developers favor the use of Apache Avro, which is a serialization framework originally developed for Hadoop. Avro provides a compact serialization format, schemas that are separate from the message payloads and that do not require code to be generated when they change, and strong data typing and schema evolution, with both backward and forward compatibility.

Using a consistent data format allows, writing, and reading messages to be decoupled.  By using well-defined schemas and storing them in a common repository, the messages in Kafka can be understood without coordination.



