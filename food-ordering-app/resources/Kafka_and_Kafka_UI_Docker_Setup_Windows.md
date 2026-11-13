# Running Apache Kafka 4.4.0 and Kafka UI with Docker on Windows

This guide documents the exact setup used in the PowerShell session
under:

`D:\SkyNetix\Microservices`

It explains what each command did, why the configuration is needed, how
the containers communicate, and how to verify the setup.

------------------------------------------------------------------------

## 1. What we are setting up

We are running two Docker containers:

  -----------------------------------------------------------------------------------------------
  Component         Docker image                      Purpose           Windows access
  ----------------- --------------------------------- ----------------- -------------------------
  Apache Kafka      `apache/kafka:4.4.0`              Message broker    `localhost:9092`

  Kafka UI          `provectuslabs/kafka-ui:latest`   Browser-based     `http://localhost:8080`
                                                      Kafka             
                                                      administration UI 
  -----------------------------------------------------------------------------------------------

Both containers are attached to a Docker network named `kafka-net`.

Kafka is configured with two client listeners:

-   **INTERNAL** --- `kafka:19092`, used by other containers on
    `kafka-net`, including Kafka UI.
-   **EXTERNAL** --- `localhost:9092`, used by applications running
    directly on Windows, such as a locally launched Spring Boot
    application.

Kafka also has a **CONTROLLER** listener on port `9093` for its KRaft
controller role. This port is internal to the Docker network in this
setup and is not published to Windows.

### Architecture

``` text
Windows host
   |
   | http://localhost:8080
   v
Kafka UI container
   |
   | kafka:19092 (Docker network)
   v
Kafka container
   |
   | localhost:9092 (published to Windows)
   v
Spring Boot application running on Windows
```

The `kafka` hostname works between containers because Docker provides
name resolution on the shared user-defined network.

------------------------------------------------------------------------

## 2. Prerequisites

Install and start:

1.  Docker Desktop for Windows.
2.  Docker Desktop's Linux container engine.
3.  PowerShell 7 or Windows PowerShell.

Check Docker:

``` powershell
docker --version
docker info
```

The commands below use PowerShell syntax.

------------------------------------------------------------------------

## 3. Create a Docker network

Command executed:

``` powershell
docker network create kafka-net
```

Output received:

``` text
cbf367531d2f9aa13fcfba64751182e1c4af3552262a8905a129b5ecfbd40519
```

### What this does

It creates a user-defined Docker bridge network named `kafka-net`.

Why use a dedicated network?

-   Kafka and Kafka UI can communicate using container names.
-   Kafka UI can connect to Kafka using `kafka:19092`.
-   We do not need to use a container IP address, which can change.
-   Other services can later join the same network if needed.

Check the network:

``` powershell
docker network ls
```

Inspect it:

``` powershell
docker network inspect kafka-net
```

------------------------------------------------------------------------

## 4. Start Apache Kafka 4.4.0

Command executed:

``` powershell
docker run -d --name kafka --network kafka-net -p 9092:9092 `
  -e KAFKA_NODE_ID=1 `
  -e KAFKA_PROCESS_ROLES=broker,controller `
  -e KAFKA_LISTENERS=INTERNAL://:19092,EXTERNAL://:9092,CONTROLLER://:9093 `
  -e KAFKA_ADVERTISED_LISTENERS=INTERNAL://kafka:19092,EXTERNAL://localhost:9092 `
  -e KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT,CONTROLLER:PLAINTEXT `
  -e KAFKA_INTER_BROKER_LISTENER_NAME=INTERNAL `
  -e KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER `
  -e KAFKA_CONTROLLER_QUORUM_VOTERS=1@kafka:9093 `
  -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 `
  -e KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR=1 `
  -e KAFKA_TRANSACTION_STATE_LOG_MIN_ISR=1 `
  apache/kafka:4.4.0
```

The original command was entered as one long line in PowerShell. The
line breaks above are only for readability; the trailing PowerShell
backtick (\`) continues a command onto the next line.

Container ID returned:

``` text
bc3a42e9ee8563feece63523ff69bcf005c706ea0dc4643d2a1d68cbba7c69ed
```

### Explanation of the command

  Part                    Meaning
  ----------------------- -----------------------------------------------------
  `docker run`            Creates and starts a container.
  `-d`                    Runs it in the background (detached mode).
  `--name kafka`          Names the container `kafka`.
  `--network kafka-net`   Connects it to the shared Docker network.
  `-p 9092:9092`          Publishes container port 9092 on Windows port 9092.
  `apache/kafka:4.4.0`    Uses Apache Kafka image version 4.4.0.

### Kafka environment variables

**KRaft configuration**

  -----------------------------------------------------------------------------------
  Variable                            Value                   Purpose
  ----------------------------------- ----------------------- -----------------------
  `KAFKA_NODE_ID`                     `1`                     Unique ID of this Kafka
                                                              node.

  `KAFKA_PROCESS_ROLES`               `broker,controller`     This single node acts
                                                              as both broker and
                                                              KRaft controller.

  `KAFKA_CONTROLLER_LISTENER_NAMES`   `CONTROLLER`            Identifies the listener
                                                              used for controller
                                                              communication.

  `KAFKA_CONTROLLER_QUORUM_VOTERS`    `1@kafka:9093`          Defines the single
                                                              controller voter: node
                                                              1 at hostname `kafka`,
                                                              port 9093.
  -----------------------------------------------------------------------------------

Kafka 4.4.0 is running in KRaft mode here. ZooKeeper is not part of this
setup.

**Listener configuration**

  --------------------------------------------------------------------------------------------------------------------------
  Variable                                 Value                                                     Purpose
  ---------------------------------------- --------------------------------------------------------- -----------------------
  `KAFKA_LISTENERS`                        `INTERNAL://:19092,EXTERNAL://:9092,CONTROLLER://:9093`   Tells Kafka which
                                                                                                     interfaces and ports to
                                                                                                     listen on.

  `KAFKA_ADVERTISED_LISTENERS`             `INTERNAL://kafka:19092,EXTERNAL://localhost:9092`        Tells clients which
                                                                                                     addresses to use after
                                                                                                     the initial connection.

  `KAFKA_LISTENER_SECURITY_PROTOCOL_MAP`   All three mapped to `PLAINTEXT`                           Maps each listener name
                                                                                                     to its Kafka security
                                                                                                     protocol.

  `KAFKA_INTER_BROKER_LISTENER_NAME`       `INTERNAL`                                                Uses the internal
                                                                                                     listener for
                                                                                                     broker-to-broker
                                                                                                     communication.
  --------------------------------------------------------------------------------------------------------------------------

The two advertised addresses solve the original Kafka UI connection
problem:

-   A container on `kafka-net` must use `kafka:19092`.
-   A program running on Windows must use `localhost:9092`.

Do not configure Kafka UI to use `localhost:9092`: inside the Kafka UI
container, `localhost` means the Kafka UI container itself, not the
Kafka container.

**Single-node replication settings**

  --------------------------------------------------------------------------------------------------
  Variable                                           Value                   Purpose
  -------------------------------------------------- ----------------------- -----------------------
  `KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR`           `1`                     Allows the internal
                                                                             consumer-offset topic
                                                                             to work on one broker.

  `KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR`   `1`                     Sets the transaction
                                                                             state topic replication
                                                                             factor to one.

  `KAFKA_TRANSACTION_STATE_LOG_MIN_ISR`              `1`                     Allows the transaction
                                                                             state topic to operate
                                                                             with one in-sync
                                                                             replica.
  --------------------------------------------------------------------------------------------------

These values are appropriate for a local, single-broker learning
environment. They are not a production high-availability configuration.

------------------------------------------------------------------------

## 5. Start Kafka UI

Command executed:

``` powershell
docker run -d --name kafka-ui --network kafka-net -p 8080:8080 `
  -e DYNAMIC_CONFIG_ENABLED=true `
  -e KAFKA_CLUSTERS_0_NAME=local `
  -e KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=kafka:19092 `
  provectuslabs/kafka-ui:latest
```

The original command was entered as one long line.

Container ID returned:

``` text
ac7c1133e5c7f944a0c8900aae56f2a449f1006658883c340dc138e7d57de161
```

### Explanation

  Part                              Meaning
  --------------------------------- --------------------------------------------
  `docker run`                      Creates and starts the Kafka UI container.
  `-d`                              Runs it in the background.
  `--name kafka-ui`                 Names the container `kafka-ui`.
  `--network kafka-net`             Connects it to the same network as Kafka.
  `-p 8080:8080`                    Publishes Kafka UI on Windows port 8080.
  `provectuslabs/kafka-ui:latest`   Uses the Kafka UI image.

### Kafka UI environment variables

  -------------------------------------------------------------------------------------
  Variable                              Value                   Purpose
  ------------------------------------- ----------------------- -----------------------
  `DYNAMIC_CONFIG_ENABLED`              `true`                  Enables dynamic cluster
                                                                configuration in Kafka
                                                                UI.

  `KAFKA_CLUSTERS_0_NAME`               `local`                 Gives the configured
                                                                cluster the display
                                                                name `local`.

  `KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS`   `kafka:19092`           Tells Kafka UI how to
                                                                reach the Kafka broker
                                                                from inside Docker.
  -------------------------------------------------------------------------------------

Kafka UI is a web application. It does not replace Kafka; it connects to
Kafka and displays broker, topic, message, and consumer-group
information.

Open it in a browser:

<http://localhost:8080>

------------------------------------------------------------------------

## 6. Verify that both containers are running

Command executed:

``` powershell
docker ps
```

Output received:

``` text
CONTAINER ID   IMAGE                           COMMAND                  CREATED          STATUS          PORTS                    NAMES
ac7c1133e5c7   provectuslabs/kafka-ui:latest   "/bin/sh -c 'java --…"   9 seconds ago    Up 8 seconds    0.0.0.0:8080->8080/tcp   kafka-ui
bc3a42e9ee85   apache/kafka:4.4.0              "/__cacert_entrypoin…"   19 seconds ago   Up 19 seconds   0.0.0.0:9092->9092/tcp   kafka
```

This confirms that Docker reports both containers as running and that
the expected host ports are published.

Important: `docker ps` alone does not prove that Kafka has finished
starting or that Kafka UI can successfully query the broker. Perform the
tests below as well.

### Check Kafka logs

``` powershell
docker logs kafka --tail 100
```

### Check Kafka UI logs

``` powershell
docker logs kafka-ui --tail 100
```

### Check Kafka broker API

``` powershell
docker exec -it kafka /opt/kafka/bin/kafka-broker-api-versions.sh --bootstrap-server localhost:9092
```

If the broker API command returns supported API/version information
rather than a connection error, the broker is responding to the client.

### Check the Docker network

``` powershell
docker network inspect kafka-net
```

Under `Containers`, verify that both `kafka` and `kafka-ui` are
attached.

------------------------------------------------------------------------

## 7. Create the first Kafka topic

After Kafka is ready, create the topic for the United Airlines learning
project:

``` powershell
docker exec -it kafka /opt/kafka/bin/kafka-topics.sh --create --topic flight-disruption-events --bootstrap-server localhost:9092
```

Expected output is similar to:

``` text
Created topic flight-disruption-events.
```

List topics:

``` powershell
docker exec -it kafka /opt/kafka/bin/kafka-topics.sh --list --bootstrap-server localhost:9092
```

Describe the topic:

``` powershell
docker exec -it kafka /opt/kafka/bin/kafka-topics.sh --describe --topic flight-disruption-events --bootstrap-server localhost:9092
```

The topic should also appear in Kafka UI under **Topics**. Refresh the
page if necessary.

------------------------------------------------------------------------

## 8. Test producing and consuming a message

Open a PowerShell terminal and start a console consumer:

``` powershell
docker exec -it kafka /opt/kafka/bin/kafka-console-consumer.sh --topic flight-disruption-events --from-beginning --bootstrap-server localhost:9092
```

Open a second PowerShell terminal and start a producer:

``` powershell
docker exec -it kafka /opt/kafka/bin/kafka-console-producer.sh --topic flight-disruption-events --bootstrap-server localhost:9092
```

Type a sample event in the producer terminal:

``` json
{"eventId":"evt-1001","eventType":"FlightDisruptionDetected","flightNumber":"UA245","reason":"WEATHER","origin":"SFO","destination":"ORD"}
```

Press Enter. The consumer terminal should display the message.

This verifies basic message production and consumption. It is a simple
text/JSON demonstration; it does not yet validate a Spring Boot
application or a production event schema.

------------------------------------------------------------------------

## 9. Connection addresses to remember

  Client location                             Bootstrap server
  ------------------------------------------- ------------------
  Kafka UI container                          `kafka:19092`
  Spring Boot running directly on Windows     `localhost:9092`
  Kafka CLI executed inside Kafka container   `localhost:9092`

If Spring Boot is later containerized and attached to `kafka-net`,
configure it to use `kafka:19092` instead of `localhost:9092`.

------------------------------------------------------------------------

## 10. Troubleshooting

### Kafka UI displays a loading spinner or broker error

Inspect the UI logs:

``` powershell
docker logs kafka-ui --tail 100
```

If the log says it cannot connect to `localhost:9092`, check that the
Kafka UI environment variable is set to:

``` text
KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=kafka:19092
```

Check the container configuration:

``` powershell
docker inspect kafka-ui
```

### Kafka container is running but not ready

Check:

``` powershell
docker logs kafka --tail 100
```

Then retry the broker API command:

``` powershell
docker exec -it kafka /opt/kafka/bin/kafka-broker-api-versions.sh --bootstrap-server localhost:9092
```

### View all containers, including stopped ones

``` powershell
docker ps -a
```

### Stop the containers

``` powershell
docker stop kafka-ui kafka
```

### Start the existing containers again

``` powershell
docker start kafka kafka-ui
```

### Remove the containers

``` powershell
docker rm -f kafka-ui kafka
```

Removing containers can remove data held only in their writable
container layers. This setup did not specify a named volume or bind
mount for Kafka data. For persistent local Kafka data, add a properly
configured volume before relying on the broker for data you need to
keep.

### Remove the Docker network

Only do this after stopping and removing the containers attached to it:

``` powershell
docker network rm kafka-net
```

------------------------------------------------------------------------

## 11. What this setup is---and is not

This is a local development and learning setup:

-   One Kafka node.
-   One KRaft controller (combined with the broker).
-   PLAINTEXT listeners without authentication or TLS.
-   Single-replica internal topics.
-   Kafka UI exposed on local port 8080.
-   No persistent Docker volume configured in the commands above.
-   No ZooKeeper.

Do not use these settings as-is for a production deployment. Production
requires a security, persistence, replication, availability, monitoring,
and operational design appropriate to the environment.

------------------------------------------------------------------------

## 12. Next steps for the United Airlines microservices exercise

Once the broker and UI are verified:

1.  Create `flight-disruption-events`.
2.  Create a Spring Boot Flight Operations producer.
3.  Create a Spring Boot Seat Inventory consumer.
4.  Publish and consume a `FlightDisruptionDetected` event.
5.  Add the Passenger Re-accommodation service.
6.  Add the Transactional Outbox pattern.
7.  Implement Saga choreography using domain events and compensating
    actions.

The first milestone is complete when the broker API responds, Kafka UI
can display the `local` cluster, and a producer message appears in the
consumer.

------------------------------------------------------------------------

## 13. Exact commands used in the successful container-start attempt

These are the commands entered in the PowerShell session, kept as single
lines:

``` powershell
docker network create kafka-net
```

``` powershell
docker run -d --name kafka --network kafka-net -p 9092:9092 -e KAFKA_NODE_ID=1 -e KAFKA_PROCESS_ROLES=broker,controller -e KAFKA_LISTENERS=INTERNAL://:19092,EXTERNAL://:9092,CONTROLLER://:9093 -e KAFKA_ADVERTISED_LISTENERS=INTERNAL://kafka:19092,EXTERNAL://localhost:9092 -e KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT,CONTROLLER:PLAINTEXT -e KAFKA_INTER_BROKER_LISTENER_NAME=INTERNAL -e KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER -e KAFKA_CONTROLLER_QUORUM_VOTERS=1@kafka:9093 -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 -e KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR=1 -e KAFKA_TRANSACTION_STATE_LOG_MIN_ISR=1 apache/kafka:4.4.0
```

``` powershell
docker run -d --name kafka-ui --network kafka-net -p 8080:8080 -e DYNAMIC_CONFIG_ENABLED=true -e KAFKA_CLUSTERS_0_NAME=local -e KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=kafka:19092 provectuslabs/kafka-ui:latest
```

``` powershell
docker ps
```

The reported output showed both containers in `Up` status. Confirm
broker readiness and Kafka UI connectivity using the verification steps
in Section 6.
