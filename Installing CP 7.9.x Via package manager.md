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
---

### Running CP Services (zookeeper, kafka, schema registry, kafka connect, ksqldb, kafka rest, C3)

1. Running zookeeper
```
sudo systemctl start confluent-zookeeper
sudo systemctl status confluent-zookeeper
```
<img width="1905" height="599" alt="image" src="https://github.com/user-attachments/assets/7528abf3-3450-4ab5-a7ee-8d9bd72b30c1" />

2. Running Kafka (broker)
```
sudo systemctl start confluent-server
sudo systemctl status confluent-server
```
<img width="1907" height="625" alt="image" src="https://github.com/user-attachments/assets/d289e512-851b-4078-9fb6-06016b3660dc" />

3. Running Schema Registry
```
sudo systemctl start confluent-schema-registry
sudo systemctl status confluent-schema-registry
```
<img width="1903" height="725" alt="image" src="https://github.com/user-attachments/assets/21a02b65-7710-440e-9f6c-ec7d073a1dec" />

4. Running Kafka Connect
```
sudo systemctl start confluent-kafka-connect
sudo systemctl status confluent-kafka-connect
```
<img width="1919" height="761" alt="image" src="https://github.com/user-attachments/assets/b15d6820-df79-497c-a1f7-1830ff1c0f65" />

5. Running ksqlDB
```
sudo systemctl start confluent-ksqldb
sudo systemctl status confluent-ksqldb
```
<img width="1909" height="618" alt="image" src="https://github.com/user-attachments/assets/4d159712-2071-4f9e-8f68-bcd5c0ce9341" />

6. Running Kafka Rest
```
sudo systemctl start confluent-kafka-rest
sudo systemctl status confluent-kafka-rest
```
<img width="1919" height="751" alt="image" src="https://github.com/user-attachments/assets/1048ded2-2ece-469d-b811-2ee67cf0130e" />

7. Running Confluent control center (C3)
```
sudo systemctl start confluent-control-center
sudo systemctl status confluent-control-center
```
<img width="1909" height="735" alt="image" src="https://github.com/user-attachments/assets/c49d7b75-aa49-44a4-b54f-4985f068beeb" />

<img width="1491" height="941" alt="image" src="https://github.com/user-attachments/assets/bf039de3-0072-44fe-b0f4-125b03ed06c1" />

---

## Testing Create Topic, Produce dan Consume Data via CLI

1. Create Topic
```
kafka-topics --create \
  --topic e2e-test \
  --partitions 3 \
  --replication-factor 1 \
  --bootstrap-server localhost:9092
```
<img width="981" height="184" alt="image" src="https://github.com/user-attachments/assets/72d771b9-3b98-40c7-925d-99af35cd4f54" />

- Verify
```
kafka-topics --describe --topic e2e-test --bootstrap-server localhost:9092
```
<img width="1841" height="397" alt="image" src="https://github.com/user-attachments/assets/545777ac-cbdb-410a-ba19-d96d582136ff" />

2. Produce Data
```
kafka-console-producer \
  --topic e2e-test \
  --bootstrap-server localhost:9092

# Send messages:
Message 1
Message 2
Message 3
(Ctrl+C to stop - jangan close)
```
<img width="1047" height="264" alt="image" src="https://github.com/user-attachments/assets/8a367ef1-a918-4fb2-a699-5860be9f8498" />

3. Consume Data
```
kafka-console-consumer \
  --topic e2e-test \
  --bootstrap-server localhost:9092 \
  --from-beginning \
  --group test-group-1

# Output: 
# Message 1
# Message 2
# Message 3
```
<img width="960" height="318" alt="image" src="https://github.com/user-attachments/assets/4e5f43a4-7dff-4179-9025-db142c981abf" />

**Monitoring dari C3**
<img width="1898" height="927" alt="image" src="https://github.com/user-attachments/assets/989bf7a9-2214-4033-b298-4c9126ae4bf2" />

---

## Pengecekan Zookeeper Quorum dan Kafka cluster id

Sebelumnya kita setup zookeper dengan single node karena memang testing dalam single server (wsl2 - ubuntu 24), sekarang kita coba scaling 3 node dalam single server dengan cara menerapkan setup port berbeda untuk masing-masing zookeeper.

1. Matikan semua service confluent
```
sudo systemctl stop confluent-zookeeper
sudo systemctl stop confluent-server
sudo systemctl stop confluent-schema-registry
sudo systemctl stop confluent-kafka-connect
sudo systemctl stop confluent-kafka-rest
sudo systemctl stop confluent-ksqldb
sudo systemctl stop confluent-control-center
```
2. Setup 3 ZK directories + myid + configs
```
# Buat 3 folder
sudo mkdir -p /var/lib/zookeeper{1,2,3}
sudo chown cp-kafka:confluent /var/lib/zookeeper{1,2,3}

# Create myid files
echo "1" | sudo tee /var/lib/zookeeper1/myid
echo "2" | sudo tee /var/lib/zookeeper2/myid
echo "3" | sudo tee /var/lib/zookeeper3/myid

# Create 3 config files
sudo tee /etc/kafka/zookeeper1.properties << 'EOF'
dataDir=/var/lib/zookeeper1
clientPort=2181
tickTime=2000
initLimit=5
syncLimit=2
autopurge.snapRetainCount=3
autopurge.purgeInterval=1
server.1=localhost:2888:3888
server.2=localhost:2889:3889
server.3=localhost:2890:3890
EOF

sudo tee /etc/kafka/zookeeper2.properties << 'EOF'
dataDir=/var/lib/zookeeper2
clientPort=2182
tickTime=2000
initLimit=5
syncLimit=2
autopurge.snapRetainCount=3
autopurge.purgeInterval=1
server.1=localhost:2888:3888
server.2=localhost:2889:3889
server.3=localhost:2890:3890
EOF

sudo tee /etc/kafka/zookeeper3.properties << 'EOF'
dataDir=/var/lib/zookeeper3
clientPort=2183
tickTime=2000
initLimit=5
syncLimit=2
autopurge.snapRetainCount=3
autopurge.purgeInterval=1
server.1=localhost:2888:3888
server.2=localhost:2889:3889
server.3=localhost:2890:3890
EOF
```

3. remove metadata broker lama
```
rm -rf /var/lib/kafka*
```
4. Update Kafka config (zookeeper.connect string)
```
sudo sed -i 's/zookeeper.connect=.*/zookeeper.connect=localhost:2181,localhost:2182,localhost:2183/' \
  /etc/kafka/server.properties
```
5. Start ZK1, ZK2, ZK3 (tunggu fully started). nb: saat awal start 1zk akan terlihat error, lanjuntkan saja start zk lainnya. setelah ketiga zk running akan terlihat normal
```
sudo /usr/bin/zookeeper-server-start /etc/kafka/zookeeper1.properties
sudo /usr/bin/zookeeper-server-start /etc/kafka/zookeeper2.properties
sudo /usr/bin/zookeeper-server-start /etc/kafka/zookeeper3.properties
```
6. cek zk berhasil start dan listen
```
sudo netstat -tlnp | grep -E "2181|2182|2183"
```
<img width="1140" height="143" alt="image" src="https://github.com/user-attachments/assets/4190c7b3-5bbd-44f9-8469-4e3d2923abff" />

7. Start Kafka Broker (tunggu fully started)
8. Start semua service confluent lainnya yg diinginkan

<img width="420" height="470" alt="image" src="https://github.com/user-attachments/assets/6d15d1b9-60ad-480c-99e6-06e091c438e4" />

9. Edit config broker agar semua info metrics muncul pada c3, lalu restart broker dan refresh ui c3
```
sudo nano /etc/kafka/server.properties
```
```
# metrick agar info pada c3 tampil
metric.reporters=io.confluent.metrics.reporter.ConfluentMetricsReporter
confluent.metrics.reporter.bootstrap.servers=localhost:9092
# Uncomment the following line if the metrics cluster has a single broker
confluent.metrics.reporter.topic.replicas=1
```

<img width="785" height="944" alt="image" src="https://github.com/user-attachments/assets/0cfb1e4c-0032-4d47-b08b-2c8514f3af17" />

**10. Cek Zookeeper quorum dan broker id**
Quorum adalah mayoritas node, dihitung dengan floor(N/2)+1. Sistem masih berjalan selama jumlah node aktif ≥ quorum
```
zookeeper-shell localhost:2181
```

```
ls /
ls /brokers/ids
get /zookeeper/config
get /controller
```

<img width="1553" height="531" alt="image" src="https://github.com/user-attachments/assets/81512655-5d69-4b46-9f10-d82ceccfca7d" />



---
















Source: 
- https://docs.confluent.io/platform/7.9/installation/installing_cp/deb-ubuntu.html#systemd-ubuntu-debian-install
- https://docs.confluent.io/platform/7.9/monitor/metrics-reporter.html
