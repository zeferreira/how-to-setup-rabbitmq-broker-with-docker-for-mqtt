# How to setup RabbitMQ Broker for MQTT/AMQP using Docker Compose for development

This is a configuration for a development environment. Don't use it for production environment. 
It works for Windows and Linux Environments (tested with windows).

The scenario is simple, the RabbitMQ broker will start without peristence and with basic authentication for one user (or more).
After it, we will test the local environment with common mtqq clients and tools (you can choose witch one you want to use).

## The test scenario:
 - One client will Subscribe the topic home/bedroom/temperature. 
 - One Client will Publish one message with the value (42 - meaning of the life) to the same topic.

## Obs.:
 - I tested it on Windows running without WSL. 
 - I tested some commands from outside the container (from the host)<!--(curl and MQTT Explorer)-->. 
 
<!-- ** I wrote one little C# application to test the flow with the broker. Not yet, but I'm working on it' -->
 - I'm using **Visual Studio Code (VScode)** because I like to see the highlight sintax of the files and I already have it on my machine, but you can use whatever you want to **create the configuration/password files** . Choose one text editor make the samples for shell commands more simple when I created this file. For god sake, give me a break. It's just a text editor.

You can download it [here](https://code.visualstudio.com/download) from the oficial website. 
Teach about how to install the VSCode is out of the scope of this document.

## 1. Docker environment

Docker installation is out of the scope of this document, as well as the commands and it's sintax. Please, check the links below if you need to understand basic concepts about docker or if you don't have Docker on your machine.

### Docker Desktop / Docker Compose 

Latest intructions are [here] (https://docs.docker.com/desktop/) on docker website.

Latest intructions are [here] (https://docs.docker.com/compose/) on docker website.

**Make sure that you understand what is a container, how to run, restart or delete a container and what is a docker compose file and how to start/stop it. You also need to understand the sintax for docker compose files, the relation between the folders on your host machine and the path that you are going to provide inside the compose file. 

It's also important the image that we are going to use and you always need to check the instructions on docker hub. Do your homework.

If you don't know the basic concepts about docker compose files check it [here](https://docs.docker.com/compose/gettingstarted/). 

### Windows Install

Latest instructions are [here](https://docs.docker.com/desktop/install/windows-install/) on docker website.

### Linux Install

Latest instructions are [here](https://docs.docker.com/desktop/install/linux-install/) on docker website.
  
## RabbitMQ Broker

I supose that you know what is MQTT protocol, a Broker and (more specificaly) Rabbit Broker. But, if you don't know it (really? And are you here?) these links below will help you:

 - Introduction about MQTT Protocol [here] (https://mqtt.org/getting-started/) on mqtt.org website.

 - Main page of RabbitMQ [here] (https://www.rabbitmq.com/) on rabbitmq.com website.

 -  What is a Message Broker [here](https://www.ibm.com/topics/message-brokers#:~:text=A%20message%20broker%20is%20software,messages%20between%20formal%20messaging%20protocols.) by IBM.

<!-- difference between mosquitto and rabbitmq -->
Before we start working on the container configuration, we need to talk about RabbitMQ. 
The most important question here is: 

## Why rabbitmq broker for MQTT?

Rabbitmq was not initially designed for MQTT and (of course) we have many options built just for the MQTT protocol. But if you want to design a system for the real world, you won't just use one protocol (mqtt) and you'll probably want to choose an architecture that is flexible enough to adapt to new requirements or scale in different ways. But you also want to reduce the complexity and cost of the solution.

I know it seems like an obvious explanation, but many good design decisions are underestimated and go unnoticed in the short term.

For example:

If you choose to work with the Mosquitto broker instead of Rabbitmq, you will only have support for MQTT. It's a simple decision, right? Not exactly. If you want to work with Mosquitto and have a microservices architecture (the hype of the moment, right?) you will probably need to add another broker to deal with microservices using another protocol for message passing (like AMQP) increasing the cost of the solution (more infrastructure, specific people to deal with a specific stack).

Another consequence is that you need to create and scale another service to read and store information from the MQTT broker and send events to the AMQP broker. Tools created to work with MQTT do not work with other protocols (such as drivers and clients). Again, the initial decision to use a specific broker for a specific protocol will not only increase complexity but also increase the cost of the solution and maintenance.

On the other hand, if you choose to work with RabbitMQ you will only have one cluster running and only one stack to deal with. The development team does not need to deal with requirements from different brokers and the team's common experience/vocabulary will be the same. Communication is very fast just because everyone has the same experience with the same tool.

Communication is really underrated these days. Even when DDD and domain specialists working with the team, you always have one specific information that create a misunderstanding when domain specialists talk with developers and you will face the same situation with a big team using different interfaces and stacks. 

And RabbitMQ wasn't designed for MQTT, so there's a trade-off if you choose to work with it. The protocol implementation is basically an emulation so it is necessary to understand what is implemented and what is not (QOS levels, for example). You also need to pay attention to application performance in some situations (more levels in the same mqtt topic) and specific management tools.

On the first moment, you will add complexity when you choose to adapt your broker to another protocol, but on the long term you just need to keep it running and people already know how to deal with it.

Another important thing is the infrastructure. Most of the projects today will start running on the Cloud. AWS, AZURE, doesn't matter. You need to check the support for your solution on the cloud provider and the tools that you have when you choose a protocol (like MQTT or AMQP) and the broker (RabbitMQ or Mosquitto).

You have good support for RabbitMQ on Azure, AWS and Google Cloud. Not just for the broker, but also for MQTT and AMQP. 

You can find information about the support to MQTT inside Cloud providers here:

 - [Azure](https://learn.microsoft.com/en-us/azure/iot/iot-mqtt-connect-to-iot-hub) 
 - [AWS](https://docs.aws.amazon.com/pt_br/iot/latest/developerguide/mqtt.html)
 - [Google](https://cloud.google.com/architecture/connected-devices/mqtt-broker-architecture)  

There is no silver bullet. Your project will define the requirements for your architecture.

Now is the time to setup the envirmonment for the container.

## 2. Create the folder for RabbitMQ configuration files

RabbitMQ doesn't enable support for MQTT by default, so we need to provide one way to choose the configuration for MQTT Plugin. 

On your host machine, open the shell (cmd or bash) and create the folders:

### Windows 

** Windows Commmand Shel

```shell

md rabbitmq_mqtt
cd rabbitmq_mqtt

# for storing rabbitmq.conf 
md config 

```

### Linux

** Linux Commmand Shel

```bash

md rabbitmq_mqtt
cd rabbitmq_mqtt

# for storing rabbitmq.conf 
md config 

```

** Pay attention to this folders, you will use it inside the docker compose to expose the files with the initial configuration.

## 3. Create RabbitMQ configuration file - rabbitmq.conf 

RabbitMQ has a configuration file named **rabbitmq.conf** and you need to create the file before you test our scenario (or you can change it later and restart it, just in case of a mistake). You can use this file to choose the MQTT Plugin configuration for your broker. 

In our case, we will choose the ports that will listen to the clients, if we accept anonymous or authenticated users and the username and password to access our server (remember our scenario).

If you want to know more about RabbitMQ MQTT Plugin configuration options check [here](https://www.rabbitmq.com/docs/mqtt) on the official website.  

Make sure that you are creating the file inside the **config folder** that we created:

### Windows / Linux

** Windows Commmand Shell

```shell
# just if you are not on the right folder 
cd rabbitmq_mqtt
cd config 
#create the empty configuration file and open it on VSCode 
code rabbitmq.conf

```

Copy and past the content below inside the file. If you read the section about the configuration options, it's basically self explanatory. We are basically telling the server that we don't want to accept connections from unauthenticated clients, the user name and password for MQTT connections, ports that we want to use to listen for new connections, the virtual host inside RabbitMQ server and the default Exchange to receive MQTT messages.

Don't worry about the Exchange, I will explain on the next session.

Basic configuration file content 
```
mqtt.listeners.tcp.default = 1883
web_mqtt.tcp.port = 15675

mqtt.allow_anonymous  = false
mqtt.default_user = guest
mqtt.default_pass = guest

mqtt.vhost            = /
mqtt.exchange         = amq.topic
mqtt.prefetch         = 10

# 24 hours by default
# mqtt.subscription_ttl = 8640000 is not supported after version 3.13
mqtt.max_session_expiry_interval_seconds = 86400

```

## 4. RabbitMQ MQTT Plugin and the overview of the MQTT Protocol inside RabbitMQ

As you know RabbitMQ was not designed to work with MQTT and the current implementation needs to "emulate" the MQTT model using the current RabbitMQ design. To understand the features implemented and how we can use RabbitMQ with both protocols, we need to understand the basic concepts behind the protocols.

 - MQTT: As you know, the MQTT model is simple a publisher sends a message to a topic and the active consumers read the message. The topic structure is divided using "/". Ex.: TopicLevel1/TopicLevel2/TopicLevel3. In our scenario we have a Subscriber listening (consuming) to the topic "home/bedroom/temperature" and a Publisher sending (publishing) messages to the same topic. 


 - AMQP: AMQP has a different model, You have a Publisher publishing (sending) Messages to an Exchange, the server apply binding rules to the Message and "route" it to Queues and Consumers will Consume (read) the queue. You also have different types of Exchanges and binding rules. This structure is part of the AMQ Model. Other players that use AMQP don't necessarily implement this AMQ Model structure and sometimes they implement just the AMQP Protocol. You could find more information about the protocol [here](https://www.amqp.org/resources/download) or [here](https://www.rabbitmq.com/tutorials/amqp-concepts). 

 Now we need to understand how we can emulate the MQTT model using AMQP.

### The MQTT Model inside RAbbitMQ

From the client perspective (Publisher/subscriber) there is no difference between the way that we Publish or subscribe to a MQTT topic. We just need to subscribe (on the consumer) and publish a message (producer). The Rabbit MQ broker creates the abstraction layer for us using the Default Exchange as "universal" topic and using MQTT topic as a routing key. If you want to publish information from MQTT client to MQTT client, you don't need to deal with it. Just set the default MQTT Exchange name inside the configuration file (we did it, remember? "mqtt.exchange = amq.topic").

You also need to avoid "." (dots) on the name of your MQTT topics because AMQP uses "." dots as separator for AMQP structure. When you send "bedroom/temperature" from your MQTT client, RabbitMQ translate it to "bedroom.temperature" behind the scenes.

This is important because if you want to send one message from one MQTT client and deliver that message to a AMQP Subscriber you need to create a biding rule to one AMQP Queue using the translated routing key "bedroom.temperature" as a routing key from the Default MQTT Exchange (amq.topic) that we created. 

You can see more details about that MQTT implementation inside rabbitMQ [here](https://www.rabbitmq.com/docs/mqtt#implementation). 

## 5. Create docker-compose file called 'docker-compose.yml'

Now, we have the folder structure to receive RabbitMQ information and the basic file to setup the server. (rabbitmq.conf inside the config folder). We also have installed the docker and docker compose and we understand the basic concept behind it. 

We also have one single text editor (VSCode) that keeps open each single file the we created, so it's easy to create another one and test it.

Keep it in mind and let's create the file.

### Windows / Linux

```shell
# just if you are not on the right folder 
cd rabbitmq_mqtt #yes, just rabbitmq_mqtt, not config.

#create the empty docker compose file and open it on VSCode 
code docker-compose.yml
```

### Content of Docker compose file

Copy and paste the content below into your **docker-compose.yml** file. The file is very simple, we are just using the broker's official image, exposing the protocol ports (these numbers are standard for AMQP/MQTT) and mapping the folder/file structure that we created previously.

Some details you could pay attention to:
- We exposed the same ports that we configured inside the configuration file. Without it, well, you know, you have no 'doors', right? With it, the container will accept connections on these ports and the service inside the container will also accept connections on the same ports. If you are running other containers on your docker, you need to check if your host has a service that is already using these ports.

- We are using the default RabbitMQ persistence model. Unlike our mosquitto configuration file ([other compose](https://github.com/zeferreira/how-to-setup-mosquitto-broker-with-docker)) we don't have a session with the configuration for persistence. RabbitMQ has a complex model with different options to store information for different versions and Queue types, for the our scenario we don't need to change it but, if you want to take a look at the persistence model and configuration you could take a look [here](https://www.rabbitmq.com/docs/persistence-conf). 

- The command "rabbitmq-plugins" is used to enable the plugins that we need. (We use the Command session inside docker compose file to do it). You can see that we want to enable MQTT, the management interface and AMQP. [More](https://www.rabbitmq.com/docs/plugins).

- The structure of folders and files correspond to the structure created previously.

```yml

version: "3.7"

services:
  rabbitmq:
    image: rabbitmq:3-management
    hostname: rabbitmq_mqtt
    container_name: rabbitmq_mqtt
    restart: unless-stopped

    ports:
      - "5672:5672"   #AMQP
      - "15672:15672" #management plugin default port
      - "1883:1883"   #MQTT
      - "9001:9001"   #MQTT
      - "15675:15675" #WEB MQTT
    volumes:
      - ./config/rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf
    
    command: "/bin/bash -c \"rabbitmq-plugins enable --offline rabbitmq_management rabbitmq_mqtt rabbitmq_web_mqtt rabbitmq_amqp1_0; rabbitmq-server\""

```

## 6. Create and run docker container with RabbitMQ broker

### Windows 

```shell
# just if you are not on the right folder 
cd rabbitmq_mqtt #yes, just rabbitmq_mqtt, not config.

# Running the docker container 
docker compose up rabbitmq_mqtt

```
### Linux

```bash
# just if you are not on the right folder 
cd rabbitmq_mqtt #yes, just rabbitmq_mqtt, not config.

# Running the docker container 
sudo docker compose up rabbitmq_mqtt

```

You also could use:

```shell
# just if you are not on the right folder 
cd rabbitmq_mqtt #yes, just rabbitmq_mqtt, not config.

# Running the docker container 
docker-compose up -d

```


### Check container status 

### Windows 

```shell

docker ps

```

### Linux


```bash

sudo docker ps

```
You must see status Up :)


## 7. User Authentication and the basic about MQTT Users inside RabbitMQ    
<!-- 
As you know, our use case has a server with authentication. So we need to choose the username and password for our mosquitto server.
We need to use mosquitto_passwd tool for it. Don't worry, Mosquitto comes with a set of tools and one of them is mosquitto_passwd.
To use it, we need to access the container to get access to the tool or just execute the it using docker exec with the right arguments:

Now, remember that the pwfile is empty, so we need to add users to our configuration file using the command:

### Windows 

```shell
docker exec mosquitto mosquitto_passwd -b /etc/mosquitto/passwd guest guest

```

### Linux 

```shell
sudo docker exec mosquitto mosquitto_passwd -b /etc/mosquitto/passwd guest guest

```
In our case, we created the user guest with the guest password. If you want to add more than one user, it's a good opportunity to learn how to connect to the container. The idea is the same, you need to execute a program that will provide access to send commands to the container. The shell.

The nexte session will provide commands to connect to the container and create more users. You can skip this session if you want.

## X.1 Connecting to the container to create more users

This is simple, call the terminal (sh) using docker exec and use the same tool to create so many users you want.  

### Windows 

```shell

#calling SH (mosquitto is our container name, please, pay attention)
docker exec -it mosquitto sh

# now you don't need to use docker exec, just call the mosquitto tool to create the new user
mosquitto_passwd -b /etc/mosquitto/passwd username2 passw2

#repeat the process every time that you need to create one user
# type 'exit' when you finish

#restart the container (mosquitto is the name of the container)
docker restart mosquitto

```
## Linux 

For linux you basically need to add **sudo** for each docker command. -->

## 8. Testing the environment 

Ok, now you have everything that you need to start using the broker. It's time to test our setup using some tools (clients) and we are done. You can use it to develop your mqtt aplications. 

### A little bit about MQTT and the test use case

For our testing scenario, we basically want to send a message using one client and receive the same message using another client. Simple.

Communication is asynchronous. The message is sent to the broker and the broker delivers the message to the second program. One program doesn't know the other one.

The idea here is that you have a program "waiting" for new messages to be processed and the broker is a 'Carrier' that handles the senders for you. You could have more than one program "listening" to a topic or more than one device (with a program) sending messages to the topic. The Topic is the structure you use to receive messages within the broker. 

Some people might think that it doesn't make sense to put another program in between the other, but in real scenarios, you could work with more than one protocol (MQTT and AMQP) and your broker would separate your second program from the first. Your processing layer does not need to know all the different protocols than the device layer.

Most of the time you also need to implement Patterns to control the flow of events between services and the broker will provide the framework for this with abstraction for storage, concurrency, different protocols and it is also a good way to scale the architecture layer that hosts your events.

If you don't understand those lines above, you should take a look at the pub/sub pattern or take a look at mosquitto documentation [here](https://mosquitto.org/man/mqtt-7.html).


Remember the test scenario:
 - One client will Subscribe the topic home/bedroom/temperature. 
 - One Client will Publish one message with the value (42 - meaning of the life) to the same topic.

Our mosquitto image already have the basic tools (mosquitto_pub/mosquitto_sub) to test the broker you don't need to install them if you want to use it. 

### Let's start the program to "listen" the messages that we send to the topic. (subscriber)

Remember that we configurated our broker with authentication, so we need to call the program with the right credentials.

### Windows
```shell

# Subscribe to the topic with authentication
docker exec mosquitto mosquitto_sub -u guest -P guest -v -t 'home/bedroom/temperature'

```

### Linux 
```shell

# Subscribe to the topic with authentication
sudo docker exec mosquitto mosquitto_sub -u guest -P guest -v -t 'home/bedroom/temperature'

```
Keep the window open. 

### Let's start the program to "publish" the messages that we send to the topic. (publisher)

### Windows
```shell

# Subscribe to the topic with authentication
docker exec mosquitto mosquitto_pub -u guest -P guest  -t 'home/bedroom/temperature' -m '42'
```

### Linux 
```shell

# Subscribe to the topic with authentication
sudo docker exec mosquitto mosquitto_pub -u guest -P guest  -t 'home/bedroom/temperature' -m '42'

```

You should see: 'home/bedroom/temperature' '42' on the windows running the subscriber (mosquitto_sub).
Keep the window open and try new messages. 


## 9. That is it.

That is it. If you did every thing right you have a MQTT broker running over docker you can use it to develop your MQTT Applications.
Thanks for you attention and let me know if you have some suggestion for this document or if you found a error. If you want to send me something, open one issue or tag me on a discussion.

I really appreciate your time to read this document and hope it's usefull for you.

