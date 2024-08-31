### Configure kafka without zookeeper

### Steps:

Install Java JDK version 11

Download Apache Kafka v2.8+ from https://kafka.apache.org/downloads under Binary

Downloads

Extract the contents on Linux

Generate a cluster ID and format the storage using kafka-storage.sh

Start Kafka using the binaries

Setup the $PATH environment variables for easy access to the Kafka binaries
===============================================
Start Kafka
The first step is to generate a new ID for your cluster
`/kafka_2.13-3.0.0/bin/kafka-storage.sh random-uuid`

This returns a UUID, for example 76BLQI7sT_ql1mBfKsOk9Q

Next, format your storage directory (replace <uuid> by your UUID obtained above)

`~/kafka_2.13-3.0.0/bin/kafka-storage.sh format -t <uuid> -c ~/kafka_2.13-3.0.0/config/kraft/server.properties`

This will format the directory that is in the log.dirs in the config/kraft/server.properties file (by default /tmp/kraft-combined-logs)

Now you can launch the broker itself in daemon mode by running this command.

`/kafka_2.13-3.0.0/bin/kafka-server-start.sh ~/kafka_2.13-3.0.0/config/kraft/server.properties`






#### Secure kafka

https://dzone.com/articles/running-kafka-in-kubernetes-with-kraft-mode-and-ss

=======================================
Deployment Components

Server Keys and Certificates
The first step to enable SSL encryption is to a create public/private key pair for every server.

⚠️ The commands in this section were executed in a Docker container running the image openjdk:11.0.10-jre because it's the same Java version (Java 11) that Confluent runs. With this approach, any possible Java version-related issue is prevented.

The next commands were executed following the Confluent Security Tutorial:

Shell
1
docker run -it --rm \
2
  --name openjdk \
3
  --mount source=kafka-certs,target=/app \
4
  openjdk:11.0.10-jre


Once in the Docker container:

Shell
1
keytool -keystore kafka-0.server.keystore.jks -alias kafka-0 -keyalg RSA -genkey


Output:

Enter keystore password:  
Re-enter new password: 
What is your first and last name?
  [Unknown]:  kafka-0.kafka-headless.kafka.svc.cluster.local
What is the name of your organizational unit?
  [Unknown]:  test
What is the name of your organization?
  [Unknown]:  test
What is the name of your City or Locality?
  [Unknown]:  Liverpool
What is the name of your State or Province?
  [Unknown]:  Merseyside
What is the two-letter country code for this unit?
  [Unknown]:  UK
Is CN=kafka-0.kafka-headless.kafka.svc.cluster.local, OU=test, O=test, L=Liverpool, ST=Merseyside, C=UK correct?
  [no]:  yes


Repeating the command for each broker:

Shell
1
keytool -keystore kafka-1.server.keystore.jks -alias kafka-1 -keyalg RSA -genkey


Shell
1
keytool -keystore kafka-2.server.keystore.jks -alias kafka-2 -keyalg RSA -genkey


Create Your Own Certificate Authority (CA)
Generate a CA that is simply a public-private key pair and certificate, and it is intended to sign other certificates.

Shell
1
openssl req -new -x509 -keyout ca-key -out ca-cert -days 90


Output:

Generating a RSA private key
...+++++
........+++++
writing new private key to 'ca-key'
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:UK
State or Province Name (full name) [Some-State]:Merseyside
Locality Name (eg, city) []:Liverpool
Organization Name (eg, company) [Internet Widgits Pty Ltd]:test
Organizational Unit Name (eg, section) []:test
Common Name (e.g. server FQDN or YOUR name) []:*.kafka-headless.kafka.svc.cluster.local
Email Address []:


Add the generated CA to the clients’ truststore so that the clients can trust this CA:

Shell
1
keytool -keystore kafka.client.truststore.jks -alias CARoot -importcert -file ca-cert


Add the generated CA to the brokers’ truststore so that the brokers can trust this CA.

Shell
1
keytool -keystore kafka-0.server.truststore.jks -alias CARoot -importcert -file ca-cert
2
keytool -keystore kafka-1.server.truststore.jks -alias CARoot -importcert -file ca-cert
3
keytool -keystore kafka-2.server.truststore.jks -alias CARoot -importcert -file ca-cert


Sign the Certificate
To sign all certificates in the keystore with the CA that you generated:

Export the certificate from the keystore:

Shell
1
keytool -keystore kafka-0.server.keystore.jks -alias kafka-0 -certreq -file cert-file-kafka-0
2
keytool -keystore kafka-1.server.keystore.jks -alias kafka-1 -certreq -file cert-file-kafka-1
3
keytool -keystore kafka-2.server.keystore.jks -alias kafka-2 -certreq -file cert-file-kafka-2


Sign it with the CA:
Shell
1
openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file-kafka-0 -out cert-signed-kafka-0 -days 90 -CAcreateserial -passin pass:${ca-password}
2
openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file-kafka-1 -out cert-signed-kafka-1 -days 90 -CAcreateserial -passin pass:${ca-password}
3
openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file-kafka-2 -out cert-signed-kafka-2 -days 90 -CAcreateserial -passin pass:${ca-password}


⚠️ Don't forget to substitute ${ca-password}

Import both the certificate of the CA and the signed certificate into the broker keystore:

Shell
1
keytool -keystore kafka-0.server.keystore.jks -alias CARoot -importcert -file ca-cert
2
keytool -keystore kafka-0.server.keystore.jks -alias kafka-0 -importcert -file cert-signed-kafka-0
3
​
4
keytool -keystore kafka-1.server.keystore.jks -alias CARoot -importcert -file ca-cert
5
keytool -keystore kafka-1.server.keystore.jks -alias kafka-1 -importcert -file cert-signed-kafka-1
6
​
7
keytool -keystore kafka-2.server.keystore.jks -alias CARoot -importcert -file ca-cert
8
keytool -keystore kafka-2.server.keystore.jks -alias kafka-2 -importcert -file cert-signed-kafka-2


⚠️ The keystore and truststore files will be used to create the ConfigMap for our deployment.

ConfigMaps
Create two ConfigMaps, one for the Kafka Broker and another one for our Kafka Client.

Kafka Broker
Create a local folder kafka-ssl and copy the keystore and truststore files into the folder. In addition, create a file broker_credswith the ${ca-password}.

Your folder should look similar to this:

Shell
1
ls kafka-ssl   
2
broker_creds                  
3
kafka-0.server.truststore.jks kafka-1.server.truststore.jks kafka-2.server.truststore.jks
4
kafka-0.server.keystore.jks   kafka-1.server.keystore.jks   kafka-2.server.keystore.jks


Create the ConfigMap:

Shell
1
kubectl create configmap kafka-ssl --from-file kafka-ssl -n kafka
2
kubectl describe configmaps -n kafka kafka-ssl  


Output:

Shell
1
Name:         kafka-ssl
2
Namespace:    kafka
3
Labels:       <none>
4
Annotations:  <none>
5
​
6
Data
7
====
8
broker_creds:
9
----
10
<redacted>
11
​
12
​
13
BinaryData
14
====
15
kafka-0.server.keystore.jks: 5001 bytes
16
kafka-0.server.truststore.jks: 1306 bytes
17
kafka-1.server.keystore.jks: 5001 bytes
18
kafka-1.server.truststore.jks: 1306 bytes
19
kafka-2.server.keystore.jks: 5001 bytes
20
kafka-2.server.truststore.jks: 1306 bytes
21
​
22
Events:  <none>


Kafka Client
Create a local folder kafka-client and copy the kafka.client.truststore.jks file into the folder. In addition, create a file broker_creds with the ${ca-password} and a file client_security.properties.

Shell
1
#client_security.properties
2
security.protocol=SSL
3
ssl.truststore.location=/etc/kafka/secrets/kafka.client.truststore.jks
4
ssl.truststore.password=<redacted>


Your folder should look similar to this:

Shell
1
ls kafka-client 
2
broker_creds                client_security.properties  kafka.client.truststore.jks


Create the ConfigMap:

Shell
1
kubectl create configmap kafka-client --from-file kafka-client -n kafka
2
kubectl describe configmaps -n kafka kafka-client  


Output:

Shell
1
Name:         kafka-client
2
Namespace:    kafka
3
Labels:       <none>
4
Annotations:  <none>
5
​
6
Data
7
====
8
broker_creds:
9
----
10
<redacted>
11
​
12
client_security.properties:
13
----
14
security.protocol=SSL
15
ssl.truststore.location=/etc/kafka/secrets/kafka.client.truststore.jks
16
ssl.truststore.password=test1234
17
ssl.endpoint.identification.algorithm=
18
​
19
BinaryData
20
====
21
kafka.client.truststore.jks: 1306 bytes
22
​
23
Events:  <none>


Confluent Kafka
This yaml file deploys a Kafka cluster within a Kubernetes namespace named kafka. It defines various Kubernetes resources required for setting up Kafka in a distributed manner.

YAML
1
---
2
apiVersion: v1
3
kind: ServiceAccount
4
metadata:
5
  name: kafka
6
  namespace: kafka
7
---
8
apiVersion: v1
9
kind: Service
10
metadata:
11
  labels:
12
    app: kafka
13
  name: kafka-headless
14
  namespace: kafka
15
spec:
16
  clusterIP: None
17
  clusterIPs:
18
  - None
19
  internalTrafficPolicy: Cluster
20
  ipFamilies:
21
  - IPv4
22
  ipFamilyPolicy: SingleStack
23
  ports:
24
  - name: tcp-kafka-int
25
    port: 9092
26
    protocol: TCP
27
    targetPort: tcp-kafka-int
28
  - name: tcp-kafka-ssl
29
    port: 9093
30
    protocol: TCP
31
    targetPort: tcp-kafka-ssl
32
  selector:
33
    app: kafka
34
  sessionAffinity: None
35
  type: ClusterIP
36
---
37
apiVersion: apps/v1
38
kind: StatefulSet
39
metadata:
40
  labels:
41
    app: kafka
42
  name: kafka
43
  namespace: kafka
44
spec:
45
  podManagementPolicy: Parallel
46
  replicas: 3
47
  revisionHistoryLimit: 10
48
  selector:
49
    matchLabels:
50
      app: kafka
51
  serviceName: kafka-headless
52
  template:
53
    metadata:
54
      labels:
55
        app: kafka
56
    spec:
57
      serviceAccountName: kafka
58
      containers:
59
      - command:
60
        - sh
61
        - -exc
62
        - |
63
          export KAFKA_NODE_ID=${HOSTNAME##*-} && \
64
          export KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://${POD_NAME}.kafka-headless.kafka.svc.cluster.local:9092,SSL://${POD_NAME}.kafka-headless.kafka.svc.cluster.local:9093
65
          export KAFKA_SSL_TRUSTSTORE_FILENAME=${POD_NAME}.server.truststore.jks
66
          export KAFKA_SSL_KEYSTORE_FILENAME=${POD_NAME}.server.keystore.jks
67
          export KAFKA_OPTS="-Djavax.net.debug=all"
68
​
69
          exec /etc/confluent/docker/run
70
        env:
71
        - name: KAFKA_SSL_KEY_CREDENTIALS
72
          value: "broker_creds"
73
        - name: KAFKA_SSL_KEYSTORE_CREDENTIALS
74
          value: "broker_creds"
75
        - name: KAFKA_SSL_TRUSTSTORE_CREDENTIALS
76
          value: "broker_creds"
77
        - name: KAFKA_LISTENER_SECURITY_PROTOCOL_MAP
78
          value: "CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,SSL:SSL"
79
        - name: CLUSTER_ID
80
          value: "6PMpHYL9QkeyXRj9Nrp4KA"
81
        - name: KAFKA_CONTROLLER_QUORUM_VOTERS
82
          value: "0@kafka-0.kafka-headless.kafka.svc.cluster.local:29093,1@kafka-1.kafka-headless.kafka.svc.cluster.local:29093,2@kafka-2.kafka-headless.kafka.svc.cluster.local:29093"
83
        - name: KAFKA_PROCESS_ROLES
84
          value: "broker,controller"
85
        - name: KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR
86
          value: "3"
87
        - name: KAFKA_NUM_PARTITIONS
88
          value: "3"
89
        - name: KAFKA_DEFAULT_REPLICATION_FACTOR
90
          value: "3"
91
        - name: KAFKA_MIN_INSYNC_REPLICAS
92
          value: "2"
93
        - name: KAFKA_CONTROLLER_LISTENER_NAMES
94
          value: "CONTROLLER"
95
        - name: KAFKA_LISTENERS
96
          value: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:29093,SSL://0.0.0.0:9093
97
        - name: POD_NAME
98
          valueFrom:
99
            fieldRef:
100
              apiVersion: v1
101
              fieldPath: metadata.name
102
        name: kafka
103
        image: docker.io/confluentinc/cp-kafka:7.5.0
104
        imagePullPolicy: IfNotPresent
105
        livenessProbe:
106
          failureThreshold: 6
107
          initialDelaySeconds: 60
108
          periodSeconds: 60
109
          successThreshold: 1
110
          tcpSocket:
111
            port: tcp-kafka-int
112
          timeoutSeconds: 5
113
        ports:
114
        - containerPort: 9092
115
          name: tcp-kafka-int
116
          protocol: TCP
117
        - containerPort: 29093
118
          name: tcp-kafka-ctrl
119
          protocol: TCP
120
        - containerPort: 9093
121
          name: tcp-kafka-ssl
122
          protocol: TCP
123
        resources:
124
          limits:
125
            cpu: "1"
126
            memory: 1400Mi
127
          requests:
128
            cpu: 250m
129
            memory: 512Mi
130
        securityContext:
131
          allowPrivilegeEscalation: false
132
          capabilities:
133
            drop:
134
            - ALL
135
          runAsGroup: 1000
136
          runAsUser: 1000
137
        terminationMessagePath: /dev/termination-log
138
        terminationMessagePolicy: File
139
        volumeMounts:
140
        - mountPath: /etc/kafka/secrets/
141
          name: kafka-ssl
142
        - mountPath: /etc/kafka
143
          name: config
144
        - mountPath: /var/lib/kafka/data
145
          name: data
146
        - mountPath: /var/log
147
          name: logs
148
      dnsPolicy: ClusterFirst
149
      restartPolicy: Always
150
      schedulerName: default-scheduler
151
      securityContext:
152
        fsGroup: 1000
153
      terminationGracePeriodSeconds: 30
154
      volumes:
155
      - emptyDir: {}
156
        name: config
157
      - emptyDir: {}
158
        name: logs
159
      - name: kafka-ssl
160
        configMap: 
161
          name: kafka-ssl
162
  updateStrategy:
163
    type: RollingUpdate
164
  volumeClaimTemplates:
165
  - apiVersion: v1
166
    kind: PersistentVolumeClaim
167
    metadata:
168
      name: data
169
    spec:
170
      accessModes:
171
      - ReadWriteOnce
172
      resources:
173
        requests:
174
          storage: 10Gi
175
      storageClassName: standard
176
      volumeMode: Filesystem
177
    status:
178
      phase: Pending

The deployment we will create will have the following components:

Namespace: kafka This is the namespace within which all components will be scoped.
Service Account: kafka Service accounts are used to control permissions and access to resources within the cluster.
Headless Service: kafka-headless It exposes ports 9092 (for PLAINTEXT communication) and 9093 (for SSL traffic).
StatefulSet: kafka It manages Kafka pods and ensures they have stable hostnames and storage.
The source code for this deployment can be found in this GitHub repository.

Specifically for the SSL configurations, the next parameters were configured in the StatefulSet:

Configure the truststore, keystore, and password: 
KAFKA_SSL_KEY_CREDENTIALS KAFKA_SSL_KEYSTORE_CREDENTIALS KAFKA_SSL_TRUSTSTORE_CREDENTIALS
Configure the ports for the Kafka brokers to listen for SSL:
KAFKA_ADVERTISED_LISTENERS KAFKA_LISTENER_SECURITY_PROTOCOL_MAP KAFKA_LISTENERS
Creating the Deployment
Clone the repo:
git clone https://github.com/rafaelmnatali/kafka-k8s.git cd ssl

Deploy Kafka using the following commands:

kubectl apply -f 00-namespace.yaml kubectl apply -f 01-kafka-local.yaml 

Verify Communication Across Brokers
There should now be three Kafka brokers each running on separate pods within your cluster. Name resolution for the headless service and the three pods within the StatefulSet is automatically configured by Kubernetes as they are created,allowing for communication across brokers. See the related documentation for more details on this feature.

You can check the first pod's logs with the following command:

kubectl logs kafka-0

The name resolution of the three pods can take more time to work than it takes the pods to start, so you may see UnknownHostException warnings in the pod logs initially:

WARN [RaftManager nodeId=2] Error connecting to node kafka-1.kafka-headless.kafka.svc.cluster.local:29093 (id: 1 rack: null) (org.apache.kafka.clients.NetworkClient) java.net.UnknownHostException: kafka-1.kafka-headless.kafka.svc.cluster.local         ...

But eventually each pod will successfully resolve pod hostnames and end with a message stating the broker has been unfenced:

INFO [Controller 0] Unfenced broker: UnfenceBrokerRecord(id=1, epoch=176) (org.apache.kafka.controller.ClusterControlManager)

Create a Topic Using the SSL Endpoint
The Kafka StatefulSet should now be up and running successfully. Now we can create a topic using the SSL endpoint.

You can deploy Kafka Client using the following command:

kubectl apply -f 02-kafka-client.yaml

Check if the Pod is Running:

kubectl get pods


Output:

NAME        READY   STATUS    RESTARTS   AGE kafka-cli 1/1 Running 0 12m

Connect to the pod kafka-cli:

kubectl exec -it kafka-cli -- bash

Create a topic named test-ssl with three partitions and a replication factor of 3.

kafka-topics --create --topic test-ssl --partitions 3 --replication-factor 3 --bootstrap-server ${BOOTSTRAP_SERVER} --command-config /etc/kafka/secrets/client_security.properties Created topic test-ssl.

The environment variable BOOTSTRAP_SERVER contains the list of the brokers, therefore, we save time in typing.

List all the topics in Kafka:

kafka-topics --bootstrap-server kafka-0.kafka-headless.kafka.svc.cluster.local:9093 --list --command-config /etc/kafka/secrets/client_security.properties   test test-ssl test-test

