### Configure kafka without zookeeper

### Steps:

Install Java JDK version 11

Download Apache Kafka v2.8+ from https://kafka.apache.org/downloads under Binary

Downloads

Extract the contents on Linux

Generate a cluster ID and format the storage using kafka-storage.sh

Start Kafka using the binaries

Setup the $PATH environment variables for easy access to the Kafka binaries

Start Kafka
The first step is to generate a new ID for your cluster
`/kafka_2.13-3.0.0/bin/kafka-storage.sh random-uuid`

This returns a UUID, for example 76BLQI7sT_ql1mBfKsOk9Q

Next, format your storage directory (replace <uuid> by your UUID obtained above)

`~/kafka_2.13-3.0.0/bin/kafka-storage.sh format -t <uuid> -c ~/kafka_2.13-3.0.0/config/kraft/server.properties`

This will format the directory that is in the log.dirs in the config/kraft/server.properties file (by default /tmp/kraft-combined-logs)

Now you can launch the broker itself in daemon mode by running this command.

`/kafka_2.13-3.0.0/bin/kafka-server-start.sh ~/kafka_2.13-3.0.0/config/kraft/server.properties`

