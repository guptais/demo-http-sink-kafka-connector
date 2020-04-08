# How to use Kafka Http Sink Connector from Confluent

## [HTTP-Sink-Connector](https://docs.confluent.io/current/connect/kafka-connect-http/index.html#connect-http-connector)

### Prerequisites:

Confluent Platform is installed and services are running locally. 

Step 1: I am using the Kafka REST Proxy to demonstrate this which is already available within the kafka cluster.

Step 2: Create 2 topics using the 2 commands or you can also use Confluent Control-center is that is running on localhost:9021

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




Capturing Failed POST requests:

We can do error reporting to produce records to after each unsuccessful POST. 
There is a lag in error reporting because of the retry attempts. Retry attempts are configurable.

SMT can be used to transform the message for the http request body. There are many pre-build transformations available. 
We can also write custom Transformations.

https://www.confluent.io/blog/kafka-connect-single-message-transformation-tutorial-with-examples/

Aggregation being a stateful transformation cannot be done within SMTs. 
Complex transformations and operations that apply to multiple messages are best implemented with KSQL and Kafka Streams.

Reference:
https://docs.confluent.io/current/quickstart/index.html
https://docs.confluent.io/current/cli/command-reference/confluent-local/index.html#confluent-local
