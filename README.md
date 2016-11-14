# ELK Stack + Curator in Docker

Run the latest version of the ELK (Elasticseach, Logstash, Kibana) stack with Docker and Docker-compose.

It will give you the ability to analyze any data set by using the searching/aggregation capabilities of Elasticseach and the visualization power of Kibana.

The docker images used:

* [elasticsearch](https://hub.docker.com/r/jkris/elasticsearch/) which are essentially based on the [official](https://registry.hub.docker.com/_/elasticsearch/) image
* [logstash](https://hub.docker.com/r/jkris/logstash/) based on [official](https://registry.hub.docker.com/_/logstash/) image
* [kibana](https://hub.docker.com/r/jkris/kibana/) based on [official](https://registry.hub.docker.com/_/kibana/) image
* [curator](https://hub.docker.com/r/jkris/curator/)

# Requirements

## Setup

1. Install [Docker](http://docker.io).
2. Install [Docker-compose](http://docs.docker.com/compose/install/) **version >= 1.6**.
3. Clone this repository

## Increase max_map_count on your host (Linux)

You need to increase `max_map_count` on your Docker host:

```bash
$ sudo sysctl -w vm.max_map_count=262144
```

## SELinux

On distributions which have SELinux enabled out-of-the-box you will need to either re-context the files or set SELinux into Permissive mode in order for the ELK stack to start properly.
For example on Redhat and CentOS, the following will apply the proper context:

```bash
.-root@centos ~
-$ chcon -R system_u:object_r:admin_home_t:s0 elk-centralized-log_play/
```

# Usage

Start the ELK stack using *docker-compose*:

```bash
$ docker-compose up
```

You can also choose to run it in background (detached mode):

```bash
$ docker-compose up -d
```

Now that the stack is running, you have the ability to inject logs in it. The shipped logstash configuration allows you to send content via tcp:

```bash
$ nc localhost 5000 < /path/to/logfile.log
```

And then access Kibana UI by hitting [http://localhost:5601](http://localhost:5601) with a web browser.

*NOTE*: You'll need to inject data into logstash before being able to create a logstash index in Kibana. Then all you should have to do is to hit the create button.

See: https://www.elastic.co/guide/en/kibana/current/setup.html#connect

By default, the stack exposes the following ports:
* 5000: Logstash TCP input.
* 9200: Elasticsearch HTTP
* 9300: Elasticsearch TCP transport
* 5601: Kibana

*WARNING*: If you're using *boot2docker*, you must access it via the *boot2docker* IP address instead of *localhost*.

*WARNING*: If you're using *Docker Toolbox*, you must access it via the *docker-machine* IP address instead of *localhost*.

# Configuration

*NOTE*: Configuration is not dynamically reloaded, you will need to restart the stack after any change in the configuration of a component.

## How can I reconfigure Kibana?

The Kibana default configuration is stored in `kibana/config/kibana.yml`.

## How can I reconfigure Logstash?

The logstash configuration is stored in `logstash/config/logstash.conf`.

The folder `logstash/config` is mapped onto the container `/etc/logstash/conf.d` so you
can create more than one file in that folder if you'd like to. However, you must be aware that config files will be read from the directory in alphabetical order.

## How can I specify the amount of memory used by Logstash?

The Logstash container use the *LS_HEAP_SIZE* environment variable to determine how much memory should be associated to the JVM heap memory (defaults to 500m).

If you want to override the default configuration, add the *LS_HEAP_SIZE* environment variable to the container in the `docker-compose.yml`:

```yml
logstash:
  image: jkris/logstash:5
  command: -f /etc/logstash/conf.d/
  volumes:
    - ./logstash/config:/etc/logstash/conf.d
  ports:
    - "5000:5000"
  links:
    - elasticsearch
  environment:
    - LS_HEAP_SIZE=2048m
```

## How can I add Logstash plugins? ##

To add plugins to logstash you have to:

1. Add a RUN statement to the `logstash/Dockerfile` (ex. `RUN logstash-plugin install logstash-filter-json`)
2. Add the associated plugin code configuration to the `logstash/config/logstash.conf` file

## How can I enable a remote JMX connection to Logstash?

As for the Java heap memory, another environment variable allows to specify JAVA_OPTS used by Logstash. You'll need to specify the appropriate options to enable JMX and map the JMX port on the docker host.

Update the container in the `docker-compose.yml` to add the *LS_JAVA_OPTS* environment variable with the following content (I've mapped the JMX service on the port 18080, you can change that), do not forget to update the *-Djava.rmi.server.hostname* option with the IP address of your Docker host (replace **DOCKER_HOST_IP**):

```yml
logstash:
  image: jkris/logstash:5
  command: -f /etc/logstash/conf.d/
  volumes:
    - ./logstash/config:/etc/logstash/conf.d
  ports:
    - "5000:5000"
  links:
    - elasticsearch
  environment:
    - LS_JAVA_OPTS=-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.port=18080 -Dcom.sun.management.jmxremote.rmi.port=18080 -Djava.rmi.server.hostname=DOCKER_HOST_IP -Dcom.sun.management.jmxremote.local.only=false
```

## How can I reconfigure Elasticsearch?

The Elasticsearch container is using the shipped configuration and it is not exposed by default.

If you want to override the default configuration, create a file `elasticsearch/config/elasticsearch.yml` and add your configuration in it.

Then, you'll need to map your configuration file inside the container in the `docker-compose.yml`. Update the elasticsearch container declaration to:

```yml
elasticsearch:
  image: jkris/elasticsearch:5
  ports:
    - "9200:9200"
    - "9300:9300"
  environment:
    ES_JAVA_OPTS: "-Xms1g -Xmx1g"
  volumes:
    - ./elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
```

You can also specify the options you want to override directly in the command field:

```yml
elasticsearch:
  image: jkris/elasticsearch:5
  command: elasticsearch -Des.network.host=_non_loopback_ -Des.cluster.name: my-cluster
  ports:
    - "9200:9200"
    - "9300:9300"
  environment:
    ES_JAVA_OPTS: "-Xms1g -Xmx1g"
```

# Storage

## How can I store Elasticsearch data?

The data stored in Elasticsearch will be persisted after container reboot but not after container removal.

In order to persist Elasticsearch data even after removing the Elasticsearch container, you'll have to mount a volume on your Docker host. Update the elasticsearch container declaration to:

```yml
elasticsearch:
  image: jkris/elasticsearch:5
  ports:
    - "9200:9200"
    - "9300:9300"
  environment:
    ES_JAVA_OPTS: "-Xms1g -Xmx1g"
  volumes:
    - /path/to/storage:/usr/share/elasticsearch/data
```

This will store elasticsearch data inside `/path/to/storage`.

# Sending logs from a Java application to Elasticsearch

To use the ELK stack as centralized logging for Java applications, we need to be able to send logs generated by the application to logstash.

## Take care of the dependencies

In order to start sending logs, we need to take care of a few dependencies. As we already use logback in the Java applications, we will be using [Logback Access Module](https://mvnrepository.com/artifact/ch.qos.logback/logback-access) and [Logstash Logback Encoder](https://mvnrepository.com/artifact/net.logstash.logback/logstash-logback-encoder) to encode the log to JSON format and send them to logstash.

Add the two lines below to your gradle dependencies file (or go to the maven repo to find a format that fits whatever build mechanism you use).

```bash
compile group: 'net.logstash.logback', name: 'logstash-logback-encoder', version: '4.7'
compile group: 'ch.qos.logback', name: 'logback-access', version: '1.1.7'
```

Rebuild your project.

## Configuring Logback

The packages above allows us to send the application logs to elasticsearch through logstash simply be adding a new `appender` in the logback.xml file.

```xml
    <contextName>your-application-name</contextName>

    <appender name="STASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
		<destination>url.to.logstash.tv2.dk:5000</destination>
		<encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
			<providers>
				<mdc /> <!-- MDC variables on the Thread will be written as JSON fields -->
				<context /> <!-- Outputs entries from logback's context -->
				<version /> <!-- Logstash json format version, the @version field in the output -->
				<logLevel />
				<loggerName />
				<pattern>
					<pattern>
						{
						"app_name": "%contextName",
						"app_version": "1.0",
						"event_date": "%date{yyyyMMdd'T'HHmmss.SSSZ}",
						"millis_since_app_start": "%relative",
						"exception": "%exception{0}"
						}
					</pattern>
				</pattern>
				<threadName />
				<message />
				<logstashMarkers /> <!-- Useful so we can add extra information for specific log lines as Markers -->
				<arguments /> <!--or through StructuredArguments -->
				<stackTrace />
			</providers>
		</encoder>
	</appender>
```

*NOTE*: Remember to change the URL and port to match the logstash server.

The `<pattern>` element allows you to add customer parameters that will be sent to elasticsearch as seperate fields. In the above I have added a few fields to allow me to better filter on logs sent by this particular application. These fields will be send in the JSON data to logstash and will be searchable in elasticsearch through kibana.

* `%contextName` will give me the name of the application.
* `%date{yyyyMMdd'T'HHmmss.SSSZ}` gives me the time of the event.
* `%relative` gives the time since the application started.
* `%exception{0}` gives the first line of the stacktrace in case of an exception.

For a fill list of which data can be collected, please take a look at the [layout conversion words](http://logback.qos.ch/manual/layouts.html#conversionWord) in the logback manual.

Remember to add the appender to the `root` in logback.xml. I have named my logstash appender `STASH`:

```xml
    <root level="INFO">
		<appender-ref ref="STDOUT" />
		<appender-ref ref="STASH"/>
	</root>
```

Your Java application should now be able to send its logs to logstash whenever you use logback.

# Managing indices through Curator


Curator can help you manage your indices in elasticsearch or to clean up data. In the config file `config.yml` remember to se the hostname and port of your elasticsearch server, like in the snippet from the config below:

```bash
client:
  hosts:
    - elasticsearch
  port: 9200
```

In the action file `action_file.yml` add your actions. E.g. if you wish to delete all indices with the prefix `my-app` that are older than `14 days`:

```bash
---
actions:
  1:
    action: delete_indices
    description: >-
      Alias indices older than 14 days, with a prefix of my-app.
    options:
      continue_if_exception: False
      disable_action: False
    filters:
    - filtertype: pattern
      kind: prefix
      value: my-app
    - filtertype: age
      source: creation_date
      direction: older
      unit: days
      unit_count: 14
```

Find more information about `Actions`, `Options` and `Filters` in the [Curator documentation](https://www.elastic.co/guide/en/elasticsearch/client/curator/4.2/about.html).

## Starting the Docker container

The configuration and the frequency of how often curator is run, is handled through the `COMMAND` and `SCHEDULE` environment variables. The `COMMAND` runs curator specifying the configs. `SCHEDULE` is a cron expression of how often curator is run. Below is an example of how to start the container.

```bash
docker run -d -e SCHEDULE='30 4 * * *' -e COMMAND='curator --config /etc/curator/config.yml  /etc/curator/action_file.yml' --link <name_of_elasticsearch_container>:elasticsearch jkris/curator:5
```