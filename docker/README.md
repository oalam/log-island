LogIsland docker files
===========

Small standalone Hadoop distribution for development and testing purpose :

- Spark 1.6.0
- Elasticsearch 2.1.1
- Kibana 4.3.1
- Logstash 2.1.1
- Flume 1.6.0


This repository contains a Docker file to build a Docker image with Apache Spark, HBase, Flume & Zeppelin. 
This Docker image depends on [centos 6.7](https://github.com/CentOS/CentOS-Dockerfiles) image.

## Pull the image from Docker Repository
```
docker pull hurence/log-island:0.9.1
```

## Building the image

```sh
# build log-island
git clone https://github.com/Hurence/log-island.git
cd log-island
# jekyll build; sbt doc;  mv target/scala-2.10/api/ docs/_site/
sbt universal:packageZipTarball 
cp target/universal/log-island-*.tgz docker/


# build kafka-manager
git clone https://github.com/yahoo/kafka-manager.git
cd kafka-manager
sbt clean dist
```

The archive is generated under dist directory, 
you have to copy this file into your Dockerfile directory you can now issue

```
docker build --rm -t hurence/log-island:0.9.1 .
```

## Running the image

* if using boot2docker make sure your VM has more than 2GB memory
* in your /etc/hosts file add $(boot2docker ip) as host 'sandbox' to make it easier to access your sandbox UI
* open yarn UI ports when running container
```
docker run \
    -it \
    -p 80:80 \
    -p 9200-9300:9200-9300 \
    -p 5601:5601 \
    -p 2181:2181 \
    -p 9092:9092 \
    -p 9000:9000 \
    -p 4050-4060:4050-4060 \
    --name log-island \
    -h sandbox \
    hurence/log-island:0.9.1 bash
```
or
```
docker run -d -h sandbox hurence/log-island:0.9.1 -d
```

if you want to mount a directory from your host :        
    
    -v /Users/tom/Documents/workspace/hurence/projects/log-island/docker/mount/:/usr/local/log-island 


## LogParser: launching a parser job

Run the following command to launch a parsing job that converts the logs into events and inject them into kafka:
```
$SPARK_HOME/bin/spark-submit \
    --class com.hurence.logisland.job.LogParserJob \
    --master local[2] \
    --jars $CLASSPATH ${lib_dir}/com.hurence.logisland-*.jar \
    --input-topics $YOUR_INPUT_TOPIC_NAME \
    --output-topics $YOUR_OUTPUT_TOPIC_NAME \
    --parser $YOUR_PARSER_CLASS \
    --batch-duration 1000 \
    --block-interval 200 \
    --max-rate-per-partition 5000 \
    --multi-lines true \
    --kafka-brokers sandbox:9092
```
with :
 - input-topics = the kafka input topic list
 - output-topics = the kafka output topic list
 - parser = the full class name of your log parser (ex: com.hurence.fdj.log.MyParser)
 - batch-duration = window time (in milliseconds) for micro batch (default: 2000)
 - block-interval = window time (in milliseconds) for determining the number of partitions per batch (default: 350)
 - max-rate-per-partition = maximum rate (in messages per second) at which each Kafka partition will be read (default: unlimited)
 - kafka-brokers = kafka broker list (ex:localhost:9092, anotherhost:9092)


## EventIndexer: launching a ES indexer job

Run the following command to launch an ES indexing job to store the events into an ES container:
```
$SPARK_HOME/bin/spark-submit \
    --class com.hurence.logisland.job.EventIndexerJob \
    --master local[2] \
    --jars $CLASSPATH ${lib_dir}/com.hurence.logisland-*.jar \
    --es-host sandbox \
    --input-topics $YOUR_INPUT_TOPIC_NAME \
    --event-mapper $YOUR_MAPPER_CLASS \
    --batch-duration 1000 \
    --block-interval 200 \
    --max-rate-per-partition 10000 \
    --index-name $YOUR_INDEX_NAME \
    --kafka-brokers sandbox:9092"
```
with :
 - input-topics = the kafka input topic list
 - output-topics = the kafka output topic list
 - parser = the full class name of your log parser (ex: com.hurence.fdj.log.MyParser)
 - batch-duration = window time (in milliseconds) for micro batch (default: 2000)
 - block-interval = window time (in milliseconds) for determining the number of partitions per batch (default: 350)
 - max-rate-per-partition = maximum rate (in messages per second) at which each Kafka partition will be read (default: unlimited)
 - kafka-brokers = kafka broker list (ex:localhost:9092, anotherhost:9092)


## Spark streaming internal configuration

Internally, the spark streaming engine is launched with the following configuration:
 - "spark.streaming.backpressure.enabled" = true =>
    Enables or disables Spark Streaming's internal backpressure mechanism (since 1.5).
    This enables the Spark Streaming to control the receiving rate based on the current batch scheduling delays and processing times so that the system receives only as fast as the system
    can process.
    Internally, this dynamically sets the maximum receiving rate of receivers.
    This rate is upper bounded by the values spark.streaming.receiver.maxRate and spark.streaming.kafka.maxRatePerPartition if they are set (see below).
 - "spark.streaming.unpersist" = false => 
     Setting this to false will allow the raw data and persisted RDDs to be accessible outside the streaming application as they will not be cleared automatically. 
     Instead, it is manualy cached and unpersisted for each RDD in order to improve the overall performances.
 - "spark.ui.port" = 4050 if available, port number increased otherwise.
 
More information about the spark streaming properties configuration available here : https://spark.apache.org/docs/1.6.0/configuration.html#spark-properties


