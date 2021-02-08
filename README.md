# DeepStream AI Inference Metadata Visualization Dashboard
A project to visualize DeepStream AI inference metadata with Kibana and build dashboards. 

Inference Device: Nvidia Jetson Nano

## Technologies Leveraged
1. <b>Nvidia DeepStream SDK</b>: Streaming Analytics toolkit that provides an end-to-end AI inference pipeline both for discrete GPUs and the Jetson Edge AI devices. 
   Documentation: https://developer.nvidia.com/deepstream-sdk
   Video Tutorial: https://www.youtube.com/watch?v=Zxyf_NazIqw

2. <b>Kafka</b>: Message broker between source (DeepStream) and destination (Elastic Stack) applications. 
3. <b>Elastic Stack </b>: A stack of tools, namely Beats, Logstash, Elastic Search and Kibana, to facilitate logs monitoring, managing unstructured data, indexing, running query transactions and data visualization. It's primarily leveraged for monitoring and analytics applications. 

## Basic Workflow

## Getting DeepStream Ready
We shall be using DeepStream 5.0 in this project. Nvidia DeepStream is a 'Streaming Analytics Toolkit' that brings everything together under one roof to ease the process of Edge AI Deployments & Cloud Connectivity in sync. Its applications mainly focus on Intelligent Video Analytics (IVA) across smart-cities, retail, medical industry, and many domains.

If you don't have it in your Jetson or Nvidia GPU workstation, please download and install it. You can refer this tutorial to get familiar about its utilities. We shall be playing with my modified version of DeepStream Test 4 app. The code is available in the repository for your perusal. The app currently accepts one source as input. We will be passing video files mainly to the app inference.

## Getting AWS Ready
For this tutorial we will be leveraging AWS to run a couple of apps from Elastic Stack (Elastic Search and Kibana). Due to resource constraints with T2 Micro machine, use your local machine for running logstash. It will also give you a chance to integrate DeepStream on Jetson with a cloud architecture. However, make sure you have an AWS account ready. Below are the necessary steps to get you started with the same. 

1. Login to AWS and launch a couple of T2 Micro instances. 
2. Go to EC2 Management Dashboard to view your launched instances and fetch your IP addresses. In this tutorial, we have used public instances. If you want to launch private instances, there just need to be a minor changes in the config files. 
3. Download and install Putty (SSH Client) and PuttyGen (key generator). Download a .pem file which will serve as a layer of authentication to start your instance via putty. 
4. Convert the .pem file to .ppk using Putty Gen. Keep this file handy. It will be always required whenever you are logging in to your AWS instance with Putty.
5. Open Putty and provide your respective IP address of your AWS instance. 
6. Navigate to SSH tab. Provide your .ppk file. 
7. Go back to Session tab and click on open to launch your session. 
8. In the putty terminal, provide the username as "ec2-user". 

## Getting Kafka Ready
We need a message broker between source (DeepStream) and destination (ELK Logstash) applications. We shall be leveraging Kafka for the same that will fetch data from DeepStream, queue it as events and transmit it to Logstash. 

To setup and launch Kafka in AWS:
Don't forget to install Java
sudo yum install java
1. Login to your AWS instance with Putty where you are going to run Kafka. 

2. To install Kafka run these commands:
	```
  wget http://apachemirror.wuchna.com/kafka/2.6.0/kafka_2.13-2.6.0.tgz
	tar xvf kafka_2.13-2.6.0.tgz
  ```

3. Navigate to kafka directory. We need to add internal and external listeners. Add listensers in server.properties file available in bin/config.
``` 
listeners=INTERNAL://0.0.0.0:19092,EXTERNAL://0.0.0.0:9092
listener.security.protocol.map=INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT
advertised.listeners=INTERNAL://ec2-3-137-212-227.us-east-2.compute.amazonaws.com:19092,EXTERNAL://ec2-3-137-212-227.us-east-2.compute.amazonaws.com:9092
inter.broker.listener.name=INTERNAL
```

4. We will need to run Zookeper which will serve as the backend host for Kafka. Start Zookeeper with this command:
```
./bin/zookeeper-server-start.sh ./config/zookeeper.properties
```

5. Start kafka
```
export KAFKA_HEAP_OPTS="-Xmx500M -Xms500M"
./bin/kafka-server-start.sh ./config/server.properties
```

6. Create topic
```
bin/kafka-topics.sh --create --topic test --bootstrap-server localhost:9092
```

7. Start Kafka producer and consumer [Optional]
```
./kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test
./kafka-console-producer.sh --bootstrap-server localhost:9092 --topic test
```

8. Download kafka tool
-- Provide cluster name
-- Provide IP
-- Provide IP and port number in Advanced tab

## Setting up Logstash

Steps (Windows):
1. Download logstash zip file (v 6.8 requires Java v8) : https://www.elastic.co/downloads/past-releases/logstash-6-8-8
2. Modify the sample config file in config folder (Refer above JSON for input)
```
input {
  kafka {
    bootstrap_servers => "3.137.221.168:9092"
    topics => "test"
    }
}
filter {
      json {
        source => "message"
      }
}
output {
  elasticsearch {
    hosts => ["3.22.114.88:9200"]
  }
}

3. Launch .bat file in bin folder
.\logstash.bat -f ..\config\test.conf
```
For Linux users: 

## Setting up Elastic Search
1. Download .rpm package and install manually
```
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.9.1-x86_64.rpm
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.9.1-x86_64.rpm.sha512
shasum -a 512 -c elasticsearch-7.9.1-x86_64.rpm.sha512 
sudo rpm --install elasticsearch-7.9.1-x86_64.rpm
```

2. Modifying heap_size
```
export _JAVA_OPTIONS="-Xms500m"
export _JAVA_OPTIONS="-Xmx500m"
java -XshowSettings:vm #Pick up jvm settings
```
Modify heap size to 500m in etc/elasticsearch/jvm.options file

3. Modify host url in .yml file in etc/elasticsearch
```
network.host: 0.0.0.0
discovery.type: single-node
```

4. Start elasticsearch
```
Enable systemctl
sudo systemctl start elasticsearch.service
```

5. To check status
```
sudo systemctl status elasticsearch.service
Check in ec2publicip:9200 on any browser
```

## Setting up Kibana

1. In yum.repos.d/amzn2-extras.repo, add this:
```
[kibana-7.x]
name=Kibana repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
```
2. After the changes:
```
sudo yum install kibana
```
3. Modifying and starting Kibana

Navigate to etc/kibana
```
vi kibana.yml
```
Make these changes:
```
server.port: 5601
elasticsearch.hosts: "http://<ec2publicip>:9200"
```
4. Start Kibana
```
sudo service kibana start
# or 
sudo systemctl start kibana.service
```
Note: To stop Kibana
```
sudo service kibana stop
```
## Launching DeepStream Application


  