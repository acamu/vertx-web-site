---
title: Vertx-Kafka-UI loop
template: post.html
date: 2018-05-05
author: acamu
---

# reactive-vertx-frontend-backend-stream

The aim of the repository is to describe an asynchronous solution from the UI to the BackEnd :
- Vertx for the microservice (service subscriber and manual service producer)
- Kafka to manage stream (this is no the subject it is treated briefly)
- A simple Frontend in HTML with SocksJs websocket subscription which is a correlationID to subscribe to a specific channel


## Part One - Manage Kafka Service

-How to start kafka & Zookeeper (simple method)

Deploy Zookeeper and Kafka binaries into there extraction directory
Modify CFG file of both

start cmd for windows a batch file

    @echo off
    echo "start Zookeeper"
    start zkServer
    echo "Kafka"
    cd "path\kafka\kafka_2.11-1.1.0\bin\windows" 
    Start kafka-server-start.bat path\kafka\kafka_2.11-1.1.0\config\server.properties


Or unix style

    #!/bin/bash
    # Script to start Kafka instance
    bin/zookeeper-server-start.sh config/zookeeper.properties
    bin/kafka-server-start.sh config/server.properties
    

Or docker style :) (very efficient)

    docker run -p 2181:2181 -p 9092:9092 --env ADVERTISED_HOST=`docker-machine ip \`docker-machine active\`` --env ADVERTISED_PORT=9092 spotify/kafka

For more info please follow the Dzone Guide to start the cluster (Reference [3])


## Part Two - Write simple UI to subscribe to a channel

The simple UI is a simple HTML file as describe below

    <!DOCTYPE html>
    <html>
    <head>
        <meta charset="utf-8">
        <title>The asynchronous actions!</title>

        <script src="//cdn.jsdelivr.net/sockjs/0.3.4/sockjs.min.js"></script>
        <script src="js/vertx-eventbus.js"></script>
        <script src="js/realtime-actions.js"></script>
    </head>
        <body>
        <h3>action</h3>
        <div id="error_message"></div>
        <form>
            Current Correlation_id:
            <span id="current_correlation_id"></span>
            <br/>
            Current content:
            <span id="current_content"></span>
            <br/>
            <div>
                <label for="correlation_id">Your current correlation_id:</label>
                <input id="correlation_id" type="text">
                <input type="button" onclick="registerHandlerForUpdateFeed();" value="Subscribe">
            </div>
            <div>
                Feed:
                <textarea id="feed" rows="4" cols="50" readonly></textarea>
            </div>
        </form>
        </body>
    </html>


the custom JS file realtime-actions.js

    function loadCurrentContent(correlation_id) {
        var correlation_id = document.getElementById('correlation_id').value;
        console.log('=>' + correlation_id);

        var xmlhttp = (window.XMLHttpRequest) ? new XMLHttpRequest() : new ActiveXObject("Microsoft.XMLHTTP");
        xmlhttp.onreadystatechange = function () {
            if (xmlhttp.readyState == 4) {
                if (xmlhttp.status == 200) {
                    document.getElementById('current_content').innerHTML = 'content ' + JSON.parse(xmlhttp.responseText).content.toString();
                } else {
                    document.getElementById('current_content').innerHTML = 'Empty';
                }
            }
        };
        xmlhttp.open("GET", "http://localhost:8080/api/controllpoints/" + correlation_id);
        xmlhttp.send();
    };

    function registerHandlerForUpdateFeed() {
        var correlation_id = document.getElementById('correlation_id').value;
        console.log('=>' + correlation_id);
        document.getElementById('current_correlation_id').innerHTML = correlation_id;
        var eventBus = new EventBus('http://localhost:8080/eventbus');
        eventBus.onopen = function () {
            eventBus.registerHandler('correlationId.' + correlation_id, function (error, message) {
                //console.log(message.body);
                var obj = JSON.parse(message.body);
                var s = JSON.stringify(message.body)
                document.getElementById('current_content').innerHTML = obj;
                document.getElementById('feed').value += 'New content: ' + s + '\n';
            });
        }
    };


The eventBus vertex file

    To download


## Part Three - Write a Producer and Consumer Vertx Verticles


### MainVerticle class

    package org.acamu.vertx;

    import io.vertx.core.AbstractVerticle;
    import io.vertx.core.Vertx;
    import org.acamu.vertx.kafka.KafkaConsumerVerticle;
    import org.acamu.vertx.kafka.KafkaProducerVerticle;
    import org.acamu.vertx.rest.UserInterfaceServiceVerticle;
    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;

    public class MainVerticle extends AbstractVerticle {

        private static final Logger LOGGER = LoggerFactory.getLogger(MainVerticle.class);

        @Override
        public void start() {
            LOGGER.info("Start of My Main Verticle");

            final Vertx vertx = Vertx.vertx();
            deployVerticle(KafkaConsumerVerticle.class.getName());
            deployVerticle(KafkaProducerVerticle.class.getName());

            deployVerticle(UserInterfaceServiceVerticle.class.getName());
        }

        protected void deployVerticle(String className) {
            vertx.deployVerticle(className, res -> {
                if (res.succeeded()) {
                    System.out.printf("Deployed %s verticle \n", className);
                } else {
                    System.out.printf("Error deploying %s verticle:%s \n", className, res.cause());
                }
            });
        }
    }


### Kafka service Consumer

    package org.acamu.vertx.kafka;

    import io.vertx.core.json.Json;
    import org.acamu.vertx.domain.ControllPoint;
    import org.apache.kafka.common.serialization.StringDeserializer;
    import io.vertx.core.AbstractVerticle;
    import io.vertx.core.Future;
    import io.vertx.kafka.client.consumer.KafkaReadStream;
    import org.apache.kafka.clients.consumer.ConsumerConfig;
    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;

    import java.util.Collections;
    import java.util.Properties;

    public class KafkaConsumerVerticle extends AbstractVerticle {

        private static final Logger LOGGER = LoggerFactory.getLogger(KafkaConsumerVerticle.class);

       // private KafkaReadStream<String, String> consumer;

        @Override
        public void start(Future<Void> future) {

            LOGGER.info("Start Kafka consumer");

            final KafkaReadStream<String, String> consumer = createConsumer();

            // we are ready w/ deployment
            future.complete();
        }

        private KafkaReadStream<String, String> createConsumer() {
            LOGGER.info("Start Kafka consumer");

            Properties config = new Properties();
            config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
            config.put(ConsumerConfig.GROUP_ID_CONFIG, "mygroup2");
            config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
            config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);

            KafkaReadStream<String, String> consumer;

            consumer = KafkaReadStream.create(vertx, config);

            consumer.subscribe(Collections.singleton("websocket_bridge"), ar -> {
                if (ar.succeeded()) {
                    LOGGER.info("Subscribed");
                } else {
                    LOGGER.error("Could not subscribe: err={}", ar.cause().getMessage());
                }
            });

            consumer.handler(record -> {
                System.out.println("Processing key=" + record.key() + ",value=" + record.value() +
                        ",partition=" + record.partition() + ",offset=" + record.offset());

                ControllPoint controllPoint = Json.decodeValue(record.value(), ControllPoint.class);
                LOGGER.info("ControllPoint processed: id={}, price={}", controllPoint.getId(), controllPoint.getPrice());
                String jsonEncode = Json.encode(controllPoint);
                LOGGER.info("send =>:"+jsonEncode);
                vertx.eventBus().publish("correlationId." + controllPoint.getId(), jsonEncode);

            });

            return consumer;
        }
    }

### Kafka service Producer (for the need of the sample)

    package org.acamu.vertx.kafka;

    import io.vertx.core.AbstractVerticle;
    import io.vertx.core.Future;
    import io.vertx.core.http.HttpMethod;
    import io.vertx.core.json.Json;
    import io.vertx.core.json.JsonObject;
    import io.vertx.ext.web.Router;
    import io.vertx.ext.web.handler.BodyHandler;
    import io.vertx.ext.web.handler.ResponseContentTypeHandler;
    import io.vertx.kafka.client.producer.KafkaProducer;
    import io.vertx.kafka.client.producer.KafkaProducerRecord;
    import io.vertx.kafka.client.producer.RecordMetadata;
    import io.vertx.kafka.client.serialization.JsonObjectSerializer;
    import org.acamu.vertx.domain.ControllPoint;
    import org.apache.kafka.clients.producer.ProducerConfig;
    import org.apache.kafka.common.serialization.StringSerializer;
    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;

    import java.util.Properties;

    //sample producer : {"id":4,"content":"test content","validated":false,"price":134}
    //url to call http://localhost:8090/controllpoint
    public class KafkaProducerVerticle extends AbstractVerticle {

        private static final Logger LOGGER = LoggerFactory.getLogger(KafkaProducerVerticle.class);

        //  private KafkaProducer<String, JsonObject> producer;

        @Override
        public void start(Future<Void> future) {

            LOGGER.info("Start Kafka producer");
            final KafkaProducer<String, JsonObject> producer = createProducer();
            /*
            Create a route to call the sample kafka bean producer
            And specify the handler which accept call. In this sample only the post method is expected
             */
            Router router = Router.router(vertx);
            router.route("/controllpoint/*").handler(ResponseContentTypeHandler.create());
            router.route(HttpMethod.POST, "/controllpoint").handler(BodyHandler.create());
            router.post("/controllpoint").produces("application/json").handler(rc -> {

                //Receive body sample to create specialised bean with specific data
                LOGGER.info("body received =>"+rc.getBodyAsString());
                ControllPoint o = Json.decodeValue(rc.getBodyAsString(), ControllPoint.class);
                KafkaProducerRecord<String, JsonObject> record = KafkaProducerRecord.create("websocket_bridge", null, rc.getBodyAsJson(), 0);
                producer.write(record, done -> {
                    if (done.succeeded()) {
                        RecordMetadata recordMetadata = done.result();
                        LOGGER.info("Record sent: msg={}, destination={}, partition={}, offset={}", record.value(), recordMetadata.getTopic(), recordMetadata.getPartition(), recordMetadata.getOffset());
                        o.setId(recordMetadata.getOffset());
                        o.setContent("PROCESSING");
                    } else {
                        Throwable t = done.cause();
                        LOGGER.error("Error sent to topic: {}", t.getMessage());
                        o.setContent("REJECTED");
                    }
                    rc.response().end(Json.encodePrettily(o));
                });
            });
            vertx.createHttpServer().requestHandler(router::accept).listen(8090);

            // we are ready w/ deployment
            future.complete();
        }

        private KafkaProducer<String, JsonObject> createProducer() {
            LOGGER.info("Start Kafka consumer");

            Properties config = new Properties();
            config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
            config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
            config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonObjectSerializer.class);
            config.put(ProducerConfig.ACKS_CONFIG, "1");
            KafkaProducer producer = KafkaProducer.create(vertx, config);

            return producer;
        }
    }

### UserInterface service (For the need of the sample)


    package org.acamu.vertx.rest;

    import io.vertx.core.AbstractVerticle;
    import io.vertx.core.eventbus.EventBus;
    import io.vertx.core.http.HttpMethod;
    import io.vertx.core.json.Json;
    import io.vertx.ext.bridge.BridgeEventType;
    import io.vertx.ext.bridge.PermittedOptions;
    import io.vertx.ext.web.Router;
    import io.vertx.ext.web.RoutingContext;
    import io.vertx.ext.web.handler.BodyHandler;
    import io.vertx.ext.web.handler.ErrorHandler;
    import io.vertx.ext.web.handler.StaticHandler;
    import io.vertx.ext.web.handler.sockjs.BridgeOptions;
    import io.vertx.ext.web.handler.sockjs.SockJSHandler;
    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;

    import java.util.HashSet;
    import java.util.Set;

    //https://vertx.io/blog/real-time-bidding-with-websockets-and-vert-x/
    public class UserInterfaceServiceVerticle extends AbstractVerticle {

        private static final Logger LOGGER = LoggerFactory.getLogger(UserInterfaceServiceVerticle.class);
        
        @Override
        public void start() {
            LOGGER.info("Start of My Verticle");

    // add cors allow to localhost:8080 address
            Router router = Router.router(vertx);

            Set<String> allowedHeaders = new HashSet<>();
            allowedHeaders.add("x-requested-with");
            allowedHeaders.add("Access-Control-Allow-Origin");
            allowedHeaders.add("origin");
            allowedHeaders.add("Content-Type");
            allowedHeaders.add("accept");
            allowedHeaders.add("X-PINGARUNER");

            Set<HttpMethod> allowedMethods = new HashSet<>();
          //  allowedMethods.add(HttpMethod.GET);
              allowedMethods.add(HttpMethod.POST);
          //  allowedMethods.add(HttpMethod.DELETE);
         //   allowedMethods.add(HttpMethod.PATCH);
         //   allowedMethods.add(HttpMethod.OPTIONS);
          //  allowedMethods.add(HttpMethod.PUT);

            // * or other like "http://localhost:8080"
            router.route().handler(io.vertx.ext.web.handler.CorsHandler.create("*")
                    .allowedHeaders(allowedHeaders)
                    .allowedMethods(allowedMethods));
            //.allowCredentials(true));

            router.route("/eventbus/*").handler(eventBusHandler());

            //router.mountSubRouter("/api", apiRouter());
            router.route().failureHandler(errorHandler());
            router.route().handler(staticHandler());

            vertx.createHttpServer().requestHandler(router::accept).listen(8080);
        }

        private SockJSHandler eventBusHandler() {
            BridgeOptions options = new BridgeOptions()
                    .addOutboundPermitted(new PermittedOptions().setAddressRegex("correlationId\\.[0-9]+"));
            return SockJSHandler.create(vertx).bridge(options, event -> {
                if (event.type() == BridgeEventType.SOCKET_CREATED) {
                    LOGGER.info("A socket was created");
                }
                event.complete(true);
            });
        }
    }

## Part Four - Call test service (with postman or something like restClient)

    EndPoint : http://localhost:8090/controllpoint
    Method : POST
    Body : {"id" : 14, "content" : "test content", "validated"  :false, "price" : 134}


# ==========================================================
# FAQ

## Websocket API vs SockJS

Unfortunately, WebSockets are not supported by all web browsers. However, there are libraries that provide a fallback when WebSockets are not available. One such library is **SockJS.** SockJS starts from trying to use the WebSocket protocol. However, if this is not possible, it uses a variety of browser-specific transport protocols. SockJS is a library designed to work in all modern browsers and in environments that do not support WebSocket protocol, for instance behind restrictive corporate proxy. SockJS provides an API similar to the standard WebSocket API.

## Enable CORS:

    Set<String> allowedHeaders = new HashSet<>();
        allowedHeaders.add("x-requested-with");
        allowedHeaders.add("Access-Control-Allow-Origin");
        allowedHeaders.add("origin");
        allowedHeaders.add("Content-Type");
        allowedHeaders.add("accept");
        allowedHeaders.add("X-PINGARUNER");

        Set<HttpMethod> allowedMethods = new HashSet<>();
        allowedMethods.add(HttpMethod.GET);
        allowedMethods.add(HttpMethod.POST);
        allowedMethods.add(HttpMethod.DELETE);
        allowedMethods.add(HttpMethod.PATCH);
        allowedMethods.add(HttpMethod.OPTIONS);
        allowedMethods.add(HttpMethod.PUT);

        // * or other like "http://localhost:8080"
        router.route().handler(io.vertx.ext.web.handler.CorsHandler.create("*")
                .allowedHeaders(allowedHeaders)
                .allowedMethods(allowedMethods));


## How to subscribe to a specific channel



# References

[1] : https://vertx.io/blog/real-time-bidding-with-websockets-and-vert-x/

[2] : https://medium.com/oril/spring-boot-websockets-angular-5-f2f4b1c14cee

[3] : https://dzone.com/articles/running-apache-kafka-on-windows-os

[4] : https://kafka.apache.org/quickstart

[5] : https://hub.docker.com/r/spotify/kafka/
