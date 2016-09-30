---
authors:
- heatherf
- ianfisher
categories:
- Spring Dataflow

date: 2016-09-28T15:39:29-07:00
draft: true
short: |
  Get started with Spring Cloud Dataflow Streams by creating a custom Sink app and deploying to Pivotal Cloud Foundry.
title: Everything and the Spring Cloud Dataflow Sink
---

# Welcome to Dataflow Streams

We recently started evaluating [Spring Cloud Dataflow](https://cloud.spring.io/spring-cloud-dataflow/) for a project and were initially overwhelmed with the capabilities this system provides. Dataflow is a very powerful tool, and we found it a bit tricky to know where to get started. Through our own experimentation and discussion with the Dataflow team, we came up with a minimal setup that we think is very useful for getting started with Dataflow, specifically around [Streams](http://cloud.spring.io/spring-cloud-stream/).

In this post we will show you how to create a simple Dataflow Stream Sink and deploy it to [Pivotal Cloud Foundry](https://run.pivotal.io/). Sinks are the component that terminate Streams, so this seemed like the absolute smallest piece we could work on to get started.

For context, Streams are made up of Sources, Sinks, and (optionally) Processors. Sources are apps that output messages, Sinks are apps that input messages, and Processors go in the middle with both input and output. (Technically, [Processors are both Sources and Sinks](https://github.com/spring-cloud/spring-cloud-stream/blob/master/spring-cloud-stream/src/main/java/org/springframework/cloud/stream/messaging/Processor.java))

`[Stream] -> [Processor] -> [Sink]`

## Generate a Sink Project

The Spring Cloud Stream Initializr can be found at http://start-scs.cfapps.io/. Here, generate a project with the `Log Sink` dependency. We chose to use Gradle, so if you're using Maven you can translate as needed.

Make sure to fill out the Project Metadata as desired. Ours looked like this:

![Project Metadata](/static/images/spring-cloud-dataflow-sink/project-metadata.png)

## Update Dependencies

The Initializr will create a [build.gradle](https://github.com/iad-dev/hello-charmander-dataflow-sink/blob/master/build.gradle) file that uses the most recent version of Spring Boot, currently 1.4.1. As of September 2016, this version is not compatible with the latest release of Spring Cloud Stream Sinks, so we used Spring Boot 1.4.0 instead.

```
// build.gradle

buildscript {
	ext {
		springBootVersion = '1.4.0.RELEASE'
	}
	...
}
```

Additionally, for deploying to Cloud Foundry, you'll need a message queue binder. We used RabbitMQ since that service is easily available on [Pivotal Web Services](https://run.pivotal.io/) where we will be hosting our apps.

```
// build.gradle

dependencies {
    compile('org.springframework.cloud:spring-cloud-stream-binder-rabbit')
    ....
}
```

## Create a Custom Sink

Next, write the code for our Sink app. We decided to simply log the messages received by the Sink to keep things simple, but Sinks can be much more substantial if needed.

We created a `io.pivotal.configuration` package and created a [SinkConfiguration](https://github.com/iad-dev/hello-charmander-dataflow-sink/blob/master/src/main/java/io/pivotal/configuration/SinkConfiguration.java) class that looks like this:

```
package io.pivotal.configuration;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.stream.messaging.Sink;
import org.springframework.integration.annotation.ServiceActivator;

@EnableBinding(Sink.class)
public class SinkConfiguration {
    private static final Log logger = LogFactory.getLog(SinkConfiguration.class);

    @ServiceActivator(inputChannel=Sink.INPUT)
    public void loggerSink(String payload) {
        logger.info("Hello, Charmander. The time is: " + payload);
    }
}
```

This class binds to the Sink input message broker channel and will receive any messages posted there in the `loggerSink` method.

############################################# COME BACK AND EXPLAIN ABOVE ANNOTATIONS

## Deploy the Dataflow Server

Download and push the Cloud Foundry Dataflow Server jar found under `Spring Cloud Data Flow Server Implementations` on https://cloud.spring.io/spring-cloud-dataflow/ to Cloud Foundry. We named our server `charmander-dataflow-server` to avoid naming conflicts on PWS, but you can call yours whatever you like.

Create the Dataflow Server app:

`cf push charmander-dataflow-server -p ~/Downloads/spring-cloud-dataflow-server-cloudfoundry-1.0.0.RELEASE.jar --no-start`

Create the services your Dataflow Server will need:

```
cf create-service rediscloud 30mb redis
cf create-service cloudamqp lemur rabbit
cf create-service p-mysql 100mb my_mysql
```

Bind the services to the Dataflow Server:

```
cf bind-service charmander-dataflow-server redis
cf bind-service charmander-dataflow-server rabbit
cf bind-service charmander-dataflow-server my_mysql
```

Set the Dataflow Server environment variables:

```
cf set-env charmander-dataflow-server SPRING_CLOUD_DEPLOYER_CLOUDFOUNDRY_URL https://api.run.pivotal.io
cf set-env charmander-dataflow-server SPRING_CLOUD_DEPLOYER_CLOUDFOUNDRY_ORG {org}
cf set-env charmander-dataflow-server SPRING_CLOUD_DEPLOYER_CLOUDFOUNDRY_SPACE {space}
cf set-env charmander-dataflow-server SPRING_CLOUD_DEPLOYER_CLOUDFOUNDRY_DOMAIN cfapps.io
cf set-env charmander-dataflow-server SPRING_CLOUD_DEPLOYER_CLOUDFOUNDRY_STREAM_SERVICES rabbit
cf set-env charmander-dataflow-server SPRING_CLOUD_DEPLOYER_CLOUDFOUNDRY_USERNAME {email}
cf set-env charmander-dataflow-server SPRING_CLOUD_DEPLOYER_CLOUDFOUNDRY_PASSWORD {password}

cf set-env charmander-dataflow-server MAVEN_REMOTE_REPOSITORIES_REPO1_URL https://repo.spring.io/libs-release
```

The Cloud Foundry username and password will be used to deploy and manage the Stream apps (such as our Sink), so make sure those credentials can manage apps within your PCF Space.

Finally, restage the server to apply your changes and start the Dataflow Server:

`cf restage charmander-dataflow-server`

Once the server starts, you can see the Dataflow Server web admin UI. Ours is at http://charmander-dataflow-server.cfapps.io/dashboard

More thorough docs for setting up Dataflow on Cloud Foundry can be found here: http://docs.spring.io/spring-cloud-dataflow-server-cloudfoundry/docs/current-SNAPSHOT/reference/htmlsingle/#_deploying_on_cloud_foundry

## Deploy the Sink App

### Host the Sink Jar

We chose to host the sink jar on AWS S3, but you could use any public url or maven repo to do so. First, you will need to build the jar using gradle.

`./gradlew clean assemble`

This will create a jar file in `build/libs`.

### Install Dataflow Shell

The Dataflow Shell application is one of the main ways you will interact with and configure the Dataflow server.

Download the shell for the Dataflow Server:
`wget http://repo.spring.io/release/org/springframework/cloud/spring-cloud-dataflow-shell/1.0.0.RELEASE/spring-cloud-dataflow-shell-1.0.0.RELEASE.jar`

Run the jar to get a shell prompt:
`java -jar ~/Downloads/spring-cloud-dataflow-shell-1.0.0.RELEASE.jar`

### Register the Sink App

From the Dataflow shell, connect to your Dataflow server:
`dataflow:>dataflow config server http://charmander-dataflow-server.cfapps.io`

Then, register your sink app:
`dataflow:>app register --name char-log --type sink --uri https://link-to-your-jar.jar`

## Create the Stream
The Dataflow server has a dashboard at http://charmander-dataflow-server.cfapps.io/dashboard. Your sink `char-log` will be listed there.

In order for data to flow to your sink, you'll need a source. We'll use the ready-made timer source on the Maven Repositories.

`dataflow:>app register --name time --type source --uri maven://org.springframework.cloud.stream.app:time-source-rabbit:1.0.0.RELEASE`

On the dashboard page, you should now see two apps: `time` of type source and `hello-char-log` of type sink.

If you click on the magnifying glass for the time source, you can see a list of properties that can be set when you make a stream. We will use `time-unit` and `fixed-delay` for our stream.

There are two ways to create a stream. First, you can use the Dataflow shell:

`dataflow:>stream create --name 'char-stream' --definition 'time --time-unit=SECONDS --fixed-delay=5 | char-log'`

This follows the format `stream create --name '{NAME_OF_STREAM}' --definition '{SOURCE_APP_NAME} {OPTIONAL_SOURCE_PROPERTIES} | {SINK_APP_NAME} {OPTIONAL_SINK_PROPERTIES}'`

You can also create a stream in the Dashboard UI by clicking on the "Streams" tab and then on "Create Stream". The available sinks, sources, and transforms will be on the left of the workspace. You can drag them and then connect sources to sinks for the configuration you want. The resulting code will be shown above the flow chart. You can then type in any variable changes into that and then use the "Create Stream" button to create it. 

![Creating a Stream in the UI](/images/spring-cloud-dataflow-sink/create-stream.png)

Your stream `char-stream` and its definition will now be listed. Go ahead and click "Deploy" and then "Deploy" on the next page (we have no 'Deployment Properties' to add).

To see the new Sink and Source apps, go to your Cloud Foundry space and notice that there are two new apps starting up, with some randomly generated words in there for you. It may take a minute or so for those spring apps to boot up.

![Sink and Source Apps in CF](/images/spring-cloud-dataflow-sink/sink_and_source.png)

To see the logging that is now done by your stream, click on the app for your sink, the one ending in 'char-log', and tail its logs. It will look something like this:

![Sink Logs](/images/spring-cloud-dataflow-sink/sink_logs.png)

You can see that every 5 seconds, we are letting our Charmander know what time it is (they are forgetful creatures).

Congratulations, your Spring Cloud Dataflow Stream is working!

In review:
1. Create the custom sink, starting with the Spring Cloud Stream Initializr
2. Deploy the Dataflow Server to Cloud Foundry and set up the Dataflow Shell
3. Register the custom sink and any other sources or sinks with the server
4. Create the stream
5. Deploy the stream
