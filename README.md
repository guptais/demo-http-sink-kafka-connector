# How to use Kafka Http Sink Connector from Confluent

## [HTTP-Sink-Connector](https://docs.confluent.io/current/connect/kafka-connect-http/index.html#connect-http-connector)

Prerequisites: Confluent Platform is installed and services are running locally.


I used the docker quickstart to run a local kafka cluster with a http connector already running in it, follow the below step to do so. 

From the project directory, run the following docker command:

```
docker-compose up -d
```
Wait for a kafka cluster and connector to become healthy. You should see something like this when you do 

```
>docker-compose ps


     Name                  Command               State                         Ports                   
-------------------------------------------------------------------------------------------------------
broker            /etc/confluent/docker/run   Up             0.0.0.0:9092->9092/tcp                    
control-center    /etc/confluent/docker/run   Up             0.0.0.0:9021->9021/tcp                    
http-connector    /etc/confluent/docker/run   Up (healthy)   0.0.0.0:8083->8083/tcp, 9092/tcp          
schema-registry   /etc/confluent/docker/run   Up             0.0.0.0:8081->8081/tcp                    
zookeeper         /etc/confluent/docker/run   Up             0.0.0.0:2181->2181/tcp, 2888/tcp, 3888/tcp
```

I am using curl command to interact with the cluster using the kafka connect container in the following way:

//Get worker cluster ID, version, and git source code commit ID:
```
curl localhost:8083/ | jq
```

List all the connectors available on a kafka-connect worker:
```
curl localhost:8083/connector-plugins | jq
```

List active connectors
```
curl localhost:8083/connectors | jq
```

Load the simple http connector with the configuration in `simpleHttpSink.json`

```
curl -X POST -H "Content-Type: application/json" --data @simpleHttpSink.json http://localhost:8083/connectors
```
Now list active connectors
```
curl localhost:8083/connectors | jq

[
  "SimpleHttpSink"
]
```

Getting tasks for a connector

```
curl localhost:8083/connectors/SimpleHttpSink/tasks | jq
```
Output should resemble:
```
{
  "name": "SimpleHttpSink",
  "config": {
    "connector.class": "io.confluent.connect.http.HttpSinkConnector",
    "confluent.topic.bootstrap.servers": "localhost:9092",
    "topics": "http-messages",
    "tasks.max": "1",
    "http.api.url": "http://localhost:8080/api/messages",
    "reporter.bootstrap.servers": "localhost:9092",
    "reporter.error.topic.name": "error-responses",
    "reporter.result.topic.name": "success-responses",
    "reporter.error.topic.replication.factor": "1",
    "confluent.topic.replication.factor": "1",
    "name": "SimpleHttpSink",
    "value.converter": "org.apache.kafka.connect.storage.StringConverter",
    "reporter.result.topic.replication.factor": "1"
  },
  "tasks": [],
  "type": "sink"
}
```
You can also visit the control center running to `localhost:9021` to see the status of the cluster.

![Confluent Control Center](./images/Control_Center.png)

Step 1: I am using the Kafka REST Proxy to demonstrate this which is already available within the kafka cluster.

Step 2: Create 2 topics using the 2 commands or you can also use Confluent Control-center which is running on localhost:9021

```
kafka-topics --zookeeper zookeeper:2181 --topic jsontest.source --create --replication-factor 1 --partitions 1
kafka-topics --zookeeper zookeeper:2181 --topic jsontest.replica --create --replication-factor 1 --partitions 1
```

Step 3: Put some data in source topic by running a kafka producer

```
kafka-console-producer --broker-list localhost:10091 --topic jsontest.source
>{"foo1":"bar1"}
>{"foo2":"bar2"}
>{"foo3":"bar3"}
>{"foo4":"bar4"}
```

Step 4: Start a consumer for the destination topic

```
kafka-console-consumer --bootstrap-server localhost:10091 --topic jsontest.replica --from-beginning
```


Step 5: Load the HTTP Sink Connector
Now we submit the HTTP connector to the cp-demo connect instance:

From outside the container:

```curl -X POST -H "Content-Type: application/json" --data '{ \
"name": "http-sink", \
"config": { \
        "connector.class":"uk.co.threefi.connect.http.HttpSinkConnector", \
        "tasks.max":"1", \
        "http.api.url":"https://restproxy:8086/topics/jsontest.replica", \
        "topics":"jsontest.source", \
        "request.method":"POST", \
        "headers":"Content-Type:application/vnd.kafka.json.v2+json|Accept:application/vnd.kafka.v2+json", \
        "value.converter":"org.apache.kafka.connect.storage.StringConverter", \
        "batch.prefix":"{\"records\":[", \
        "batch.suffix":"]}", \
        "batch.max.size":"5", \
        "regex.patterns":"^~$", \
        "regex.replacements":"{\"value\":~}", \
        "regex.separator":"~" }}' \
http://localhost:8083/connectors
```
Output should resemble:

```{ \
"name":"http-sink", \
"config":{ \
        "connector.class":"uk.co.threefi.connect.http.HttpSinkConnector", \
        "tasks.max":"1", \
        "http.api.url":"https://restproxy:8086/topics/jsontest.replica", \
        "topics":"jsontest.source", \
        "request.method":"POST", \
        "headers":"Content-Type:application/vnd.kafka.json.v2+json|Accept:application/vnd.kafka.v2+json", \
        "value.converter":"org.apache.kafka.connect.storage.StringConverter", \
        "batch.prefix":"{\"records\":[", \
        "batch.suffix":"]}", \
        "batch.max.size":"5", \
        "regex.patterns":"^~$", \
        "regex.replacements":"{\"value\":~}", \
        "regex.separator":"~", \
        "name":"http-sink"}, \
"tasks":[], \
"type":null \
}
```
Kafka REST Proxy expects data to be wrapped in a structure as below:

```
{
  "records":[
        { "value": {"foo1":"bar1" } },
        { "value": {"foo2":"bar2" } }
      ]
}
```

In the opened kafka console consumer you should see the following:

```
{"foo1":"bar1"}
{"foo2":"bar2"}
{"foo3":"bar3"}
{"foo4":"bar4"}
{"foo5":"bar5"}
```

[Configuration details about the connector](https://docs.confluent.io/current/connect/kafka-connect-http/connector_config.html#connection)

#### Capturing Failed POST requests:

We can do error reporting to produce records to after each unsuccessful POST. 
There is a lag in error reporting because of the retry attempts. Retry attempts are configurable.

SMT can be used to transform the message for the http request body. There are many pre-build transformations available. 
We can also write custom Transformations.

https://www.confluent.io/blog/kafka-connect-single-message-transformation-tutorial-with-examples/

Aggregation, being a stateful transformation, cannot be done within SMTs. 
Complex transformations and operations that apply to multiple messages are best implemented with KSQL and Kafka Streams.

Reference:
https://docs.confluent.io/current/quickstart/index.html
https://docs.confluent.io/current/cli/command-reference/confluent-local/index.html#confluent-local
