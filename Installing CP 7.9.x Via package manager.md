# Installing CP 7.9.x via package manager on debian/ubuntu (wsl2 - ubuntu 24)

1. Make a directory to store the Confluent public key used to sign Confluent packages in the APT repository.
```
sudo mkdir -p /etc/apt/keyrings
```
2. Download and install the Confluent public key.
```
wget -qO - https://packages.confluent.io/deb/7.9/archive.key | gpg \
--dearmor | sudo tee /etc/apt/keyrings/confluent.gpg > /dev/null
```
3. Add the Confluent repository to /etc/apt/sources.list.d referencing the location of the signing key.
```
CP_DIST=$(lsb_release -cs)
echo "Types: deb
URIs: https://packages.confluent.io/deb/7.9
Suites: stable
Components: main
Architectures: $(dpkg --print-architecture)
Signed-by: /etc/apt/keyrings/confluent.gpg

Types: deb
URIs: https://packages.confluent.io/clients/deb/
Suites: ${CP_DIST}
Components: main
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/confluent.gpg" | sudo tee /etc/apt/sources.list.d/confluent-platform.sources > /dev/null
```
<img width="1710" height="541" alt="image" src="https://github.com/user-attachments/assets/ae3caa4c-323c-449f-aa7d-5d7893c7e682" />

4. Update apt-get and install the entire Confluent Platform package.
- Confluent Platform:
  ```
  sudo apt-get update && sudo apt-get install confluent-platform
  ```
### Configure Confluent Platform

1. Navigate to the ZooKeeper properties file (/etc/kafka/zookeeper.properties) file and modify as shown.
```
tickTime=2000
dataDir=/var/lib/zookeeper/
clientPort=2181
initLimit=5
syncLimit=2
server.1=localhost:2888:3888
# server.2=zoo2:2888:3888
# server.3=zoo3:2888:3888
autopurge.snapRetainCount=3
autopurge.purgeInterval=24
```
<img width="1448" height="863" alt="image" src="https://github.com/user-attachments/assets/ebcb21d8-7d0f-4767-beb6-3b39efd2c372" />

2.  Navigate to the Broker properties file (/etc/kafka/server.properties)
```
zookeeper.connect=localhost:2181
log.dirs=/var/lib/kafka
broker.id=0
# Listeners for metadata server
confluent.metadata.server.listeners=http://0.0.0.0:9644
# Advertised listeners for metadata server
confluent.metadata.server.advertised.listeners=http://127.0.0.1:9644
```

3. Navigate to the Schema Registry properties file (/etc/schema-registry/schema-registry.properties) and specify the following properties:
```
# Specify the address the socket server listens on, e.g. listeners = PLAINTEXT://your.host.name:9092
listeners=http://0.0.0.0:8085 #default 8081

# The host name advertised in ZooKeeper. This must be specified if your running Schema Registry
# with multiple nodes.
host.name=localhost

# List of Kafka brokers to connect to, e.g. PLAINTEXT://hostname:9092,SSL://hostname2:9092
kafkastore.bootstrap.servers=PLAINTEXT://localhost:9092
```

4.  Kafka Connect /etc/kafka/connect-distributed.properties
```
rest.port=8083
rest.advertised.host.name=localhost
rest.advertised.port=8083
plugin.path=/usr/share/java
```

5.  Kafka Rest /etc/kafka-rest/kafka-rest.properties
```
id=kafka-rest-test-server
bootstrap.servers=PLAINTEXT://localhost:9092
# ZooKeeper (optional)
zookeeper.connect=localhost:2181
# Schema Registry
schema.registry.url=http://localhost:8085
# REST Proxy port
listeners=http://0.0.0.0:8090
```

6.  ksqlDB /etc/ksqldb/ksql-server.properties
```
listeners=http://0.0.0.0:8088
rest.advertised.listener=http://localhost:8088
bootstrap.servers=localhost:9092
ksql.connect.url=http://localhost:8083
ksql.schema.registry.url=http://localhost:8085
# ksql.schema.registry.url=http://localhost:8081 (default)
state.dir=/var/lib/kafka-streams
```

7.  C3 (Confluent control center)
   - ubah env c3 dari env prod ke env dev agar tidak perlu ada lisensi
   ```
     sudo nano /usr/lib/systemd/system/confluent-control-center.service
   ```
   - Reload systemd
   ```
   sudo systemctl daemon-reload
   ```
   - ubah config env dev: /etc/confluent-control-center/control-center-dev.properties
   ```
   bootstrap.servers=localhost:9092
   zookeeper.connect=localhost:2181
   confluent.controlcenter.id=1
   confluent.controlcenter.connect.connect-default.cluster=http://localhost:8083
   confluent.controlcenter.ksql.ksqlDB.url=http://localhost:8088
   confluent.controlcenter.schema.registry.url=http://localhost:8085
   confluent.controlcenter.streams.cprest.url=http://localhost:8090
   ```
   - 














Source: https://docs.confluent.io/platform/7.9/installation/installing_cp/deb-ubuntu.html#systemd-ubuntu-debian-install
