# How to use Kafka Http Sink Connector from Confluent

## [HTTP-Sink-Connector](https://docs.confluent.io/current/connect/kafka-connect-http/index.html#connect-http-connector)

### Prerequisites:

Confluent Platform is installed and services are running locally.

### How to install the run a local kafka cluster

Note: Download the Confluent platform tar or zip and extract it into a directory `~/confluent-platform`. Set an environment variable to this path.

```
export CONFLUENT_HOME=~/confluent-platform
export PATH=$PATH:$CONFLUENT_HOME/bin
```
Install confluent hub client using brew. Commands for MacOS

```
brew tap confluentinc/homebrew-confluent-hub-client
brew cask install confluent-hub-client
```

### Install http connector using confluent hub client.

```
confluent-hub install confluentinc/kafka-connect-http:latest
```

## Demo REST API service

I am using an existing java REST API available on the internet. 
clone and run the `kafka-connect-http-demo` app on your machine.

```
git clone https://github.com/confluentinc/kafka-connect-http-demo.git
cd kafka-connect-http-demo
mvn spring-boot:run -Dspring.profiles.active=simple-auth
```

### Start the local kafka cluster

```
confluent local start
```

### Load the simple Http sink connector

```
confluent local load httpsink -- -d simpleHttpSink.json
```

### Check status of connector

```
confluent local status SimpleHttpSink
```

Produce some message to the `http-messages` topic

```
confluent local produce http-messages
```

Open few consumers in another terminal window to monitor the other topics.

```
confluent local consume <topic-name>
```

I am interested in monitoring 3 topics:
- http-messages
- success-responses
- error-responses

I have opened 5 terminals all splitted in a window, as shown below:

#### Capturing the HTTP Response from the API

I produced string messages to the Source topic and received them in the Success Response topic

![Messages received in Success response topic](./images/Success-Response-Topic.png)

#### Capturing Failed POST requests:

We can do error reporting to produce records to after each unsuccessful POST.
There is a lag in error reporting because of the retry attempts. Retry attempts are configurable.

When the REST API was down due to any error, I received the error in the Error Response topic

![Messages in Error Response topic](./images/Error-Response-topic.png)

#### Transformation of messages

SMT(Single Message Transform) can be used to transform the message for the http request body. There are many pre-build transformations available.
We can also write custom Transformations.

https://www.confluent.io/blog/kafka-connect-single-message-transformation-tutorial-with-examples/

Aggregation, being a stateful transformation, cannot be done within SMTs.
Complex transformations and operations that apply to multiple messages are best implemented with KSQL and Kafka Streams.

[Configuration details about the connector](https://docs.confluent.io/current/connect/kafka-connect-http/connector_config.html#connection)

### Troubleshooting:

Below are some commands to debug some issues:

```
curl -X POST localhost:8083/connectors/SimpleHttpSink/restart | jq

confluent local unload SimpleHttpSink -- -d simpleHttpSink.json

curl localhost:8083/connectors/SimpleHttpSink/status | jq

```
You can also delete the connector and create it again. Refer the below links.

### References:

- https://docs.confluent.io/current/quickstart/index.html
- https://docs.confluent.io/current/cli/command-reference/confluent-local/index.html#confluent-local
- https://docs.confluent.io/3.2.0/connect/managing.html
- https://www.confluent.io/blog/webify-event-streams-using-kafka-connect-http-sink/

