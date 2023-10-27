# kafka-installation

# Command uses 

### Command to  install
```docker compose -f <docker-compose.yml> up -d``` 
 
 -- -d indicates run process in background

### Move into Kafka container
```docker exec -it <kafka_conatiner_id> /bin/sh```
### Go inside kafka installation folder
```cd /opt/kafka_<version>/bin```
### Create Kafka topic
```kafka-topics.sh --create --zookeeper zookeeper:2181 --replication-factor 1 --partitions 1 --topic quickstart```
### Start Producer app (CLI)
```kafka-console-producer.sh --topic quickstart --bootstrap-server localhost:9092```
### Start consumer app (CLI)
```kafka-console-consumer.sh --topic quickstart --from-beginning --bootstrap-server localhost:9092```
![why_Kafka](https://github.com/rtls-net/kafka-installation-docker/assets/147226341/35fbe775-ef5f-4c26-ac5c-cd31d85db82a)


![kafka](https://github.com/rtls-net/kafka-installation-docker/assets/147226341/3ac1b4f3-a8d4-4e00-a032-4901f98c8af2)


![kafka components](https://github.com/rtls-net/kafka-installation-docker/assets/147226341/a45141a4-59e3-428e-834a-a3a2ed473231)


