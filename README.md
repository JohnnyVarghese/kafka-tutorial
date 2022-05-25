# kafka-tutorial
Kafka tutorial

## 1. Install the prerequisites
* [Install Docker Desktop](https://docs.docker.com/desktop/windows/install/)
* Install Brew
```bash    
    #Install Brew
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

    #Install Homebrew's dependencies
    sudo apt-get install build-essential
    brew install gcc

    #Verify brew
    brew doctor
```

## 2. Install Kafka with Zookeeper(using Docker)
```bash

    # Clone the tutorial repo
    git clone https://github.com/conduktor/kafka-stack-docker-compose.git
    cd kafka-stack-docker-compose

    #Start single broker kafka cluster
    docker-compose -f zk-single-kafka-single.yml up -d 

    #Check if Kafka and zookeeper containers are running,
    $ docker-compose -f zk-single-kafka-single.yml ps

    #Install Kafka CLI in the host machine,
    $ brew install kafka
```

## 3. Fundamentals
### Kafka Architecture
![Kafka Architecture](https://cdn.educba.com/academy/wp-content/uploads/2019/12/Kafka-Architecture-2-1.png "Kafka Architecture")

### Terminologies
1. Topic:
   * A particular stream of data
   * A topic is identified by it's name
   * Can have as many topics as we want
   * Supports any kind of message format, e.g. JSON, AVRO, binary format, etc.
   * A sequence of message is called a data stream
   * Topics cannot be queried, instead use Kafka Producers to send data and Kafka consumers to read data
   * Data is kept in Kafka only for a limited time(default is 1 week but it's configurable)
1. Partitions
   * Topics are split in paritions
   * Can have as many partitions as we want per topic
   * Messages within each partition are ordered  
   * Kafka topics are immutable: once data is written to a partition, it cannot be changed
   * Order is guaranteed only within a partition(not across partitions)
   * Data is assigned randomly to a partition unless a key is provided
1. Message offset
   * Each message within a partition gets an incremental id, called offset
   * Offset are unique only within a partition
   * Offsets are not re-used even if previous messages have been deleted
1. Producers
   * Producers write data to topics
   * Producers know to which partition to write to and which Kafka broker has the partition
   * In case of Kafka broker failures, producers will automatically recover
   * Producers can choose to send a key with the message(string, number, binary, etc.)
   * if key=null, data is sent round robin(partition 0, then 1 and then 2 and so on)
   * if key!=null, then all messages for that key will go to the same partition(hashing)
1. Producers Acknowledgements(acks)
   * Producers can choose to receive acknowledgement of data writes:
     * acks=0: Producer won't wait for acknowledgement(possible data loss)
     * acks=1: Producer will wait for leader acknowledgement(limited data loss)
     * acks=all: Leader + replicas aknowledgement(no data loss)
1. Message Anatomy
   * Key: Can be null. Format: binary
   * Value: Can be null. Format: binary
   * Compression Type: none, gzip, snappy, lz4, zstd
   * Headers(optional): List of Key-Value pairs.
   * Partition + Offset: The partition to which the message will be sent to and the offset
   * Timestamp: Either set by the system or by the user
1. Kafka Message Serializer
   * Kafka only accepts bytes as an input from producers and sends bytes out as an output to consumers
   * Message serialization means transforming objects/data into bytes
   * They are used on the value and the key
   * Common serializers: String(including JSON), Int, Float, Avro, Protobuf
1. Kafka Message Key Hashing
   * A Kafka partitioner is a code logic that takes a record and determines to which partition to send it into
   * Key Hashing is the process of determining the mapping of a key to a partition
   * In the default Kafka partitioner, the keys are hashed using the murmur2 algorithm, with the below formula:
        ```Java
            targetPartition = Math.abs(Utils.murumur2(keyBytes)) % (numPartitions - 1)
        ```
1. Consumers
   * Consumers read data from a topic(identified by name) - pull model
   * *Consumers automatically know which broker to read from*
   * *In case of broker failures, consumers know how to recover*
   * Data is read in order from low to high within each partitions
1.  Consumer Deserliazer
   * Deserliaze indicates how to transform bytes into objects/data
   * Consumer should know in advance what serializers are used on the key and value of an incoming message
   * Common Deserializers: String(including JSON), Int, Float, Avro, Protobuf
   * The serialization/deserialization type must not change during a topic lifecycle(create a new topic instead)
11. Consumer Groups
    * All the consumers in an application read data as a consumer groups
    * Each consumer within a group reads from exclusive partitions. Within a consumer group, a consumer can read from multiple topics but each topic will have only 1 consumer within 1 consumer group
    * If there are more consumers than partitions, some consumers will be inactive
    ![Consumer Groups](https://niqdev.github.io/devops/img/kafka-consumer-group.png "Consumer Groups")
1. Consumer offsets
    * Kafka stores the offsets at which a consumer group has been reading
    * The offsets commited are in Kafka topic named *__consumer_offsets*
    * When a consumer in a group has processed data received from Kafka, it should be periodically committing the offsets(the Kafka broker will write to __consumer_offsets, not the group itself). That way if a consumer dies, it will be able to read back from where it left off.
1. Delivery semantics for consumers
    * At least once(usually preferred)
      * Offsets are committed after the message is processed
      * If the processing goes wrong, the messages will be read again
      * This can result in duplicate processing of messages. Make sure your processing is *idempotent*(i.e. processing again the messages won't impact your systems)
    * At most once
      * Offsets are committed as soon as messages are received
      * If the processing goes wrong, some messages will be lost(they won't be read again)
    * Exactly once
      * *For Kafka => Kafka workflows: use the Transactional API(easy with Kafka Streams API)*
      * *For Kafka => External System workflows: use an idempotent consumer*
1. Cluster
    * A Kafka cluster is composed of multiple browsers(servers)
    * Each broker is identified with its ID(integer)
    * *Every Kafka broker is also called a "bootstrap server". That means that you only need to connect to one broker, and the Kafka clients will know how to be connected to the entire cluster(smart clients).*
    * Each broker contains certain topic partitions
    * A good number to get started is 3 brokers
1. Topic replication factor
    * Topics should have a replication factor > 1(usually between 2 and 3)
    * This way if a broker is down, another broker can serve the data
    * Concept of Leader for a Partition
      * At any time only 1 broker can be a leader for a given partition
      * Producers can only send data to the broker that is leader of a partition and consumers by default will read from the leader broker for a partition.
        
        But since Kafka v2.4, it is possible to configure consumers to read from the closest replica. This may help improve latency, and also decrease network costs if using the cloud.
      * The other brokers will replicate the data
      * There, each parition has one leader and multiple ISR(in-sync replica)
1. Kafka Topic Durability
    * For a topic replicator factor of 3, topic data durability can withstand 2 broker loss
    * As a rule, for a replication factor of N, you can permanently lose up to N-1 brokers and still recover your data
1. Zookeeper
    * Zookeeper manages brokers(keeps a list of them)
    * Zookeeper helps in performing leader election of partitions
    * Zookeeper sends notification to Kafka in case of changes(e.g. new topic, broker dies, broker comes up, delete topics, etc.)
    * Kafka 2.x can't work without Zookeeper
    * Kafka 3.x can work without Zookeeper - using Kafka Raft instead
    * Kafka 4.x will not have Zookeeper
    * Zookeeper by design operates with an odd number of servers(1,3,5,7)
    * Zookeeper has a leader(writes) the rest of servers are followers(reads)
    * Zookeeper does NOT store consumer offsets with Kafka > v0.10
1. Kafka KRaft
    * In 2020, the Apache Kafka project start to work to remove the Zookeeper dependency from it(KIP-500)
    * Zookeeper shows scaling issues when Kafka cluster have > 100,000 partitions
    * By removing Zookeeper, Apache Kafka can
      * Scale to millions of partitions, and becomes easier to maintain and set-up
      * Improve stability, makes it easier to monitor, support and administer
      * Single security model for the whole system
      * Single process to start with Kafka
      * Faster controller shutdown and recovery time
 
### CLI commands
```bash
    #Create topic
    kafka-topics --bootstrap-server localhost:9092 --create --topic firstTopic replication-factor 2 --partitions 3

    #List topics
    kafka-topics --bootstrap-server localhost:9092 --list

    #Describe topic
    kafka-topics --bootstrap-server localhost:9092 --describe --topic firstTopic

    #Describe all topics
    kafka-topics --bootstrap-server localhost:9092 --describe --topic

    #Delete topic
    kafka-topics --bootstrap-server localhost:9092 --delete --topic firstTopic


    #Send data to a topic
    kafka-console-producer --bootstrap-server localhost:9092 --topic secondTopic
    >Hello world!
    >Ctrl+c

    #Send data to a topic with a key
    kafka-console-producer --bootstrap-server localhost:9092 --topic firstTopic --property parse.key=true --property key.separator=:
    >key1:Hello world!
    >Ctrl+c


    #Consume data from the moment it's started
    kafka-console-consumer --bootstrap-server localhost:9092 --topic firstTopic

    #Consume data from the beginning
    kafka-console-consumer --bootstrap-server localhost:9092 --topic firstTopic --from-beginning

    #Consume data from the beginning and print the data using a formatter to display message timestamp, key and the value
    kafka-console-consumer --bootstrap-server localhost:9092 --topic firstTopic --formatter kafka.tools.DefaultMessageFormatter --property print.timestamp=true --property print.key=true --property print.value=true --from-beginning

    #Create consumer group
    kafka-console-consumer --bootstrap-server localhost:9092 --topic firstTopic --group firstGroup

    #List consumer groups
    kafka-consumer-groups --bootstrap-server localhost:9092 --list

    #Describe all consumer groups
    kafka-consumer-groups --bootstrap-server localhost:9092 --describe --all-groups

    #Describe a consumer group
    kafka-consumer-groups --bootstrap-server localhost:9092 --describe --group firstGroup

    #Consumer group - reset partition offsets to the earliest for all topics
    kafka-consumer-groups --bootstrap-server localhost:9092 --group firstGroup --reset-offsets --to-earliest --execute --all-topics

    #Consumer group - reset partition offsets to the earliest for a topic
    kafka-consumer-groups --bootstrap-server localhost:9092 --group firstGroup --reset-offsets --to-earliest --execute --topic firstTopic

    #Consumer group - reset partition offsets forward by 2 for a topic
    kafka-consumer-groups --bootstrap-server localhost:9092 --group firstGroup --reset-offsets --shift-by 2 --execute --topic firstTopic

    #Consumer group - reset partition offsets backward by 2 for a topic
    kafka-consumer-groups --bootstrap-server localhost:9092 --group firstGroup --reset-offsets --shift-by -2 --execute --topic firstTopic
```