# Task 2: Deploy Confluent Platform Lengkap Tanpa Security
## Automate Confluent Deployment dengan Ansible — Full Stack

> **Referensi Resmi:** [Confluent Ansible Install — No Security](https://docs.confluent.io/ansible/7.9/ansible-install.html)
> **Prasyarat:** Task 1 selesai (Ansible 9.x + `confluent.platform 7.9.5` terinstall di `cp-ansible`)

---

## Gambaran Umum

Task ini mencakup deployment **full stack Confluent Platform 7.9** menggunakan Ansible tanpa konfigurasi security (plaintext). Semua komponen ekosistem Confluent akan di-deploy secara otomatis ke cluster 3 node.

### Komponen yang Akan Di-deploy

| Komponen | Singkatan | Fungsi Utama |
|----------|-----------|--------------|
| ZooKeeper | ZK | Koordinasi cluster, leader election, metadata Kafka |
| Kafka Broker | KB | Message broker inti, menyimpan dan meneruskan event |
| Schema Registry | SR | Manajemen schema Avro/Protobuf/JSON, schema evolution |
| Kafka Connect | KC | Integrasi data in/out antara Kafka dengan sistem eksternal |
| ksqlDB | KSQL | Stream processing dengan SQL di atas Kafka |
| Kafka REST Proxy | REST | HTTP REST API untuk produce/consume Kafka |
| Control Center | C3 | UI monitoring dan manajemen cluster (non-next-gen) |

---

## Topologi Deployment

```
┌────────────────────────────────────────────────────────────────────────┐
│                    Confluent Platform 7.9 — Full Stack                 │
│                         (Tanpa Security / Plaintext)                   │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    cp-node1  (192.168.56.11)                     │  │
│  │  ZooKeeper:2181  │  Kafka:9092  │  Schema Registry:8081          │  │
│  │  Kafka Connect:8083  │  ksqlDB:8088  │  REST Proxy:8082          │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    cp-node2  (192.168.56.12)                     │  │
│  │  ZooKeeper:2181  │  Kafka:9092  │  Schema Registry:8081          │  │
│  │  Kafka Connect:8083  │  ksqlDB:8088  │  REST Proxy:8082          │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    cp-node3  (192.168.56.13)                     │  │
│  │  ZooKeeper:2181  │  Kafka:9092  │  Schema Registry:8081          │  │
│  │  Kafka Connect:8083  │  ksqlDB:8088  │  REST Proxy:8082          │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │               Control Center  (192.168.56.10 / cp-ansible)       │  │
│  │                       Port: 9021                                  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
```

> **Catatan Arsitektur:**
> Di setup ini semua service (kecuali Control Center) berjalan di setiap node untuk demonstrasi HA. Pada deployment production, Control Center biasanya dijalankan di node terpisah yang didedikasikan. Control Center di sini diinstall di `cp-ansible` sebagai management node.

### Port Reference Lengkap

| Komponen | Port | Protokol | Keterangan |
|----------|------|----------|------------|
| ZooKeeper Client | 2181 | TCP | Kafka broker connect ke ZooKeeper |
| ZooKeeper Peer | 2888 | TCP | Komunikasi follower → leader |
| ZooKeeper Leader Election | 3888 | TCP | Voting ZooKeeper leader |
| Kafka Broker | 9092 | TCP/Plaintext | Kafka producer/consumer |
| Kafka JMX | 9999 | TCP | Monitoring JMX (opsional) |
| Schema Registry | 8081 | HTTP | REST API schema management |
| Kafka Connect | 8083 | HTTP | REST API connector management |
| ksqlDB | 8088 | HTTP | REST API ksqlDB queries |
| Kafka REST Proxy | 8082 | HTTP | HTTP API produce/consume Kafka |
| Control Center | 9021 | HTTP | Web UI monitoring |

---

## Alur Deployment

```
[cp-ansible]
     │
     ├── Step 1: Siapkan struktur project directory
     ├── Step 2: Buat file inventory (hosts.yml) — semua 7 komponen
     ├── Step 3: Buat file konfigurasi platform (vars/platform.yml)
     ├── Step 4: Buat playbook utama (confluent.yml)
     ├── Step 5: Verifikasi syntax & konektivitas
     └── Step 6: Jalankan deployment
           │
           ▼
     [cp-node1, cp-node2, cp-node3]
           │
           ├── Install JDK 17
           ├── Install & konfigurasi ZooKeeper
           ├── Install & konfigurasi Kafka Broker
           ├── Install & konfigurasi Schema Registry
           ├── Install & konfigurasi Kafka Connect
           ├── Install & konfigurasi ksqlDB
           ├── Install & konfigurasi REST Proxy
           └── Install & konfigurasi Control Center (di cp-ansible)
```

---

## Step 1: Masuk ke Control Node dan Siapkan Struktur Project

Login ke `cp-ansible` dan siapkan direktori kerja:

```bash
# Login ke cp-ansible
ssh cpadmin@192.168.56.10

# Pastikan direktori confluent-ansible sudah ada dari Task 1
ls ~/confluent-ansible/
```

Buat struktur lengkap:

```bash
mkdir -p ~/confluent-ansible/{inventory,vars,certs,plugins/connectors}
cd ~/confluent-ansible
```

**Penjelasan struktur direktori:**

| Direktori | Fungsi |
|-----------|--------|
| `inventory/` | File hosts.yml yang mendefinisikan node dan peran |
| `vars/` | File konfigurasi platform (versi, properti) |
| `certs/` | Placeholder untuk certificate TLS (dipakai di Task 3) |
| `plugins/connectors/` | Direktori untuk custom Kafka Connect connector plugins |

**Verifikasi struktur:**

```bash
tree ~/confluent-ansible/
```

Output yang diharapkan:
```
/home/cpadmin/confluent-ansible/
├── certs/
├── inventory/
├── plugins/
│   └── connectors/
└── vars/
```

---

## Step 2: Buat File Inventory Lengkap

File inventory mendefinisikan **siapa saja node yang akan dikonfigurasi** dan **peran masing-masing node** dalam ekosistem Confluent Platform.

```bash
cat > ~/confluent-ansible/inventory/hosts.yml << 'EOF'
---
all:
  vars:
    # ============================================================
    # Konfigurasi koneksi SSH untuk semua node
    # ============================================================
    ansible_user: cpadmin
    ansible_become: true
    ansible_ssh_private_key_file: ~/.ssh/id_rsa

# ============================================================
# ZooKeeper — Koordinasi cluster Kafka
# 3 node membentuk ZooKeeper ensemble (quorum butuh minimal 2 node aktif)
# ============================================================
zookeeper:
  hosts:
    cp-node1:
      ansible_host: 192.168.56.11
    cp-node2:
      ansible_host: 192.168.56.12
    cp-node3:
      ansible_host: 192.168.56.13

# ============================================================
# Kafka Broker — Message broker inti
# 3 broker membentuk Kafka cluster
# ============================================================
kafka_broker:
  hosts:
    cp-node1:
      ansible_host: 192.168.56.11
    cp-node2:
      ansible_host: 192.168.56.12
    cp-node3:
      ansible_host: 192.168.56.13

# ============================================================
# Schema Registry — Manajemen schema Avro/Protobuf/JSON
# Multi-node untuk HA: satu active, lainnya standby
# ============================================================
schema_registry:
  hosts:
    cp-node1:
      ansible_host: 192.168.56.11
    cp-node2:
      ansible_host: 192.168.56.12
    cp-node3:
      ansible_host: 192.168.56.13

# ============================================================
# Kafka Connect — Integrasi data (connectors)
# Multi-node membentuk Connect cluster distributed mode
# ============================================================
kafka_connect:
  hosts:
    cp-node1:
      ansible_host: 192.168.56.11
    cp-node2:
      ansible_host: 192.168.56.12
    cp-node3:
      ansible_host: 192.168.56.13

# ============================================================
# ksqlDB — Stream processing dengan SQL
# Multi-node membentuk ksqlDB cluster
# ============================================================
ksql:
  hosts:
    cp-node1:
      ansible_host: 192.168.56.11
    cp-node2:
      ansible_host: 192.168.56.12
    cp-node3:
      ansible_host: 192.168.56.13

# ============================================================
# Kafka REST Proxy — HTTP API untuk Kafka
# Multi-node untuk load balancing dan HA
# ============================================================
kafka_rest:
  hosts:
    cp-node1:
      ansible_host: 192.168.56.11
    cp-node2:
      ansible_host: 192.168.56.12
    cp-node3:
      ansible_host: 192.168.56.13

# ============================================================
# Control Center — Web UI monitoring dan management
# Single instance (bukan next-gen), di-deploy di cp-ansible
# ============================================================
control_center:
  hosts:
    cp-ansible:
      ansible_host: 192.168.56.10
EOF
```

### Penjelasan Setiap Group Inventory

**`zookeeper`** — ZooKeeper Ensemble

ZooKeeper adalah distributed coordination service yang digunakan Kafka untuk:
- Menyimpan metadata cluster (daftar broker, topic, partisi)
- Leader election untuk Kafka controller
- Menyimpan konfigurasi consumer group (pada versi lama)
- Sinkronisasi state antar broker

Dengan 3 node, ensemble memiliki quorum saat minimal 2 node aktif. Jika hanya 1 node aktif, ZooKeeper tidak bisa menerima write (tidak ada quorum).

**`kafka_broker`** — Kafka Cluster

Setiap broker di dalam cluster memiliki peran ganda:
- **Leader** untuk beberapa partisi — menerima semua write dan read dari producer/consumer
- **Follower/Replica** untuk partisi lainnya — mereplikasi data dari leader

Dengan 3 broker dan `replication.factor=3`, data tersebar di semua node. Cluster tetap beroperasi jika 1 broker mati.

**`schema_registry`** — Schema Registry Cluster

Schema Registry menyimpan schema Avro/Protobuf/JSON-Schema. Dalam mode multi-node:
- Satu node dipilih sebagai **master** (active-master) yang menerima write
- Node lain berfungsi sebagai **slave** yang melayani read
- Jika master mati, salah satu slave otomatis dipromosikan

**`kafka_connect`** — Connect Cluster (Distributed Mode)

Kafka Connect dalam distributed mode memungkinkan:
- Connector dan task didistribusi ke seluruh worker
- Jika satu worker mati, task otomatis di-rebalance ke worker lain
- Konfigurasi connector disimpan di internal Kafka topic (`connect-configs`, `connect-offsets`, `connect-status`)

**`ksql`** — ksqlDB Cluster

ksqlDB adalah database streaming yang memungkinkan query SQL pada Kafka stream. Dalam cluster mode:
- Persistent query didistribusikan ke semua node
- State store direplikasi antar node untuk fault tolerance
- Setiap node bisa menerima request (tidak ada single master)

**`kafka_rest`** — REST Proxy

Kafka REST Proxy menyediakan RESTful HTTP API untuk:
- Produce pesan ke Kafka via HTTP POST
- Consume pesan dari Kafka via HTTP GET
- Manage consumer groups via REST
- Berguna untuk aplikasi yang tidak bisa menggunakan Kafka client library

**`control_center`** — Control Center

Control Center adalah web UI untuk monitoring dan manajemen seluruh ekosistem Confluent. Fitur utamanya:
- Dashboard realtime cluster health
- Topic explorer dan message browsing
- Consumer group lag monitoring
- Kafka Connect management
- ksqlDB query editor
- Schema Registry browser
- Alerting dan notifications

> **Catatan:** Control Center versi klasik (non-next-gen) di-deploy di `cp-ansible` agar tidak menambah beban ke node data.

**Testing Verifikasi Inventory:**

```bash
cd ~/confluent-ansible

# Tampilkan semua host dalam inventory (format JSON)
ansible-inventory -i inventory/hosts.yml --list

# Tampilkan dalam format tree
ansible-inventory -i inventory/hosts.yml --graph
```

Output `--graph` yang diharapkan:
```
@all:
  |--@zookeeper:
  |  |--cp-node1
  |  |--cp-node2
  |  |--cp-node3
  |--@kafka_broker:
  |  |--cp-node1
  |  |--cp-node2
  |  |--cp-node3
  |--@schema_registry:
  |  |--cp-node1
  |  |--cp-node2
  |  |--cp-node3
  |--@kafka_connect:
  |  |--cp-node1
  |  |--cp-node2
  |  |--cp-node3
  |--@ksql:
  |  |--cp-node1
  |  |--cp-node2
  |  |--cp-node3
  |--@kafka_rest:
  |  |--cp-node1
  |  |--cp-node2
  |  |--cp-node3
  |--@control_center:
  |  |--cp-ansible
  |--@ungrouped:
```

✅ **Jika semua group dan host muncul, inventory valid.**

---

## Step 3: Buat File Konfigurasi Platform (vars/platform.yml)

File ini adalah **pusat konfigurasi** seluruh Confluent Platform. Semua properti komponen didefinisikan di sini.

```bash
cat > ~/confluent-ansible/vars/platform.yml << 'EOF'
---
# ============================================================
# Confluent Platform Configuration
# Task 2: Full Stack — No Security (Plaintext)
# Versi: 7.9.5
# ============================================================

# Versi Confluent Platform yang akan diinstall
confluent_package_version: "7.9.5"

# Repository Confluent (pastikan network bisa akses ini)
confluent_repo_version: "7.9"

# ============================================================
# Security Configuration — DISABLED untuk Task 2
# ============================================================
# Nonaktifkan TLS/SSL encryption
ssl_enabled: false

# Nonaktifkan SASL authentication
sasl_protocol: none

# Nonaktifkan RBAC (Role-Based Access Control)
rbac_enabled: false

# ============================================================
# ZooKeeper Configuration
# ============================================================
zookeeper_custom_properties:
  # tickTime: unit waktu dasar ZooKeeper dalam milidetik.
  # Semua timeout (initLimit, syncLimit) diukur dalam kelipatan tickTime.
  tickTime: 2000

  # initLimit: waktu maksimum (dalam tick) yang diberikan kepada follower
  # untuk terhubung dan sync dengan leader saat startup cluster.
  # 10 tick × 2000ms = 20 detik. Untuk jaringan lambat, naikkan nilai ini.
  initLimit: 10

  # syncLimit: waktu maksimum (dalam tick) yang diberikan kepada follower
  # untuk sync dengan leader saat operasi normal (bukan startup).
  # 5 tick × 2000ms = 10 detik. Jika follower tidak sync dalam waktu ini,
  # follower dianggap mati dan akan di-disconnect.
  syncLimit: 5

  # maxClientCnxns: batas maksimum koneksi dari satu IP address ke ZooKeeper.
  # 0 = unlimited. Nilai 60 mencegah satu client memonopoli koneksi.
  maxClientCnxns: 60

  # Ukuran maksimum ZooKeeper transaction log sebelum snapshot dibuat (bytes)
  snapCount: 100000

  # autopurge: otomatis hapus snapshot lama untuk mencegah disk penuh.
  # snapRetainCount: jumlah snapshot terbaru yang dipertahankan.
  autopurge.snapRetainCount: 3

  # purgeInterval: seberapa sering (dalam jam) auto-purge berjalan.
  # 0 = disabled. Nilai 1 = setiap jam.
  autopurge.purgeInterval: 1

  # Batas ukuran maximum data per znode (bytes)
  jute.maxbuffer: 1048576

# ============================================================
# Kafka Broker Configuration
# ============================================================
kafka_broker_custom_properties:
  # ----------------------------------------------------------------
  # Replication Settings
  # ----------------------------------------------------------------

  # default.replication.factor: jumlah replica untuk setiap partisi
  # topic baru yang dibuat tanpa replication.factor eksplisit.
  # Nilai 3 berarti setiap partisi ada di 3 broker — tahan 1 broker failure.
  default.replication.factor: 3

  # min.insync.replicas: jumlah minimum replica yang harus "in-sync"
  # agar producer dengan acks=all/-1 bisa berhasil menulis.
  # Dengan nilai 2, cluster masih bisa write meski 1 broker mati.
  # Jika kurang dari 2 replica in-sync, producer akan mendapat NotEnoughReplicasException.
  min.insync.replicas: 2

  # offsets.topic.replication.factor: replikasi untuk internal topic
  # __consumer_offsets yang menyimpan posisi offset consumer group.
  offsets.topic.replication.factor: 3

  # transaction.state.log.replication.factor: replikasi untuk internal topic
  # __transaction_state yang digunakan untuk Kafka transactions (exactly-once).
  transaction.state.log.replication.factor: 3

  # transaction.state.log.min.isr: ISR minimum untuk transaction state log.
  transaction.state.log.min.isr: 2

  # ----------------------------------------------------------------
  # Partitions Settings
  # ----------------------------------------------------------------

  # num.partitions: jumlah partisi default untuk topic baru.
  # Lebih banyak partisi = lebih tinggi parallelism, tapi lebih banyak file.
  # Panduan umum: jumlah partisi = jumlah consumer dalam consumer group.
  num.partitions: 3

  # ----------------------------------------------------------------
  # Network & I/O Settings
  # ----------------------------------------------------------------

  # num.network.threads: thread untuk handle network request (accept + send).
  # Naikkan jika Kafka mengalami network bottleneck (tinggi network wait).
  num.network.threads: 3

  # num.io.threads: thread untuk menangani request yang butuh I/O disk.
  # Naikkan jika disk I/O menjadi bottleneck. Biasanya 2× jumlah disk.
  num.io.threads: 8

  # socket.send.buffer.bytes: ukuran send buffer socket OS untuk koneksi Kafka.
  # 102400 = 100 KB. Naikkan untuk throughput tinggi di jaringan cepat.
  socket.send.buffer.bytes: 102400

  # socket.receive.buffer.bytes: ukuran receive buffer socket OS.
  socket.receive.buffer.bytes: 102400

  # socket.request.max.bytes: ukuran maksimum request yang bisa diterima broker.
  # 104857600 = 100 MB. Ini adalah batas atas untuk message size.
  socket.request.max.bytes: 104857600

  # queued.max.requests: maksimum request yang bisa antri di I/O thread
  # sebelum network thread berhenti membaca dari socket baru.
  queued.max.requests: 500

  # ----------------------------------------------------------------
  # Log / Storage Settings
  # ----------------------------------------------------------------

  # log.retention.hours: berapa lama data disimpan sebelum dihapus.
  # 168 jam = 7 hari. Ini adalah retention berbasis waktu (mana duluan: waktu atau ukuran).
  log.retention.hours: 168

  # log.retention.bytes: ukuran maksimum total log per partisi.
  # -1 = tidak ada batas ukuran (hanya berbasis waktu).
  log.retention.bytes: -1

  # log.segment.bytes: ukuran maksimum satu file segment log (bytes).
  # 1073741824 = 1 GB. Setelah segment penuh, segment baru dibuat.
  # Kompaksi dan penghapusan hanya terjadi pada segment yang sudah "rolled".
  log.segment.bytes: 1073741824

  # log.retention.check.interval.ms: seberapa sering log cleaner mengecek
  # apakah ada log yang perlu dihapus berdasarkan retention policy.
  # 300000 ms = 5 menit.
  log.retention.check.interval.ms: 300000

  # log.cleaner.enable: aktifkan log compaction untuk topic dengan cleanup.policy=compact.
  # Diperlukan untuk Schema Registry, Connect, ksqlDB internal topics.
  log.cleaner.enable: true

  # ----------------------------------------------------------------
  # Consumer Group & Coordinator Settings
  # ----------------------------------------------------------------

  # group.initial.rebalance.delay.ms: waktu tunggu sebelum rebalance pertama
  # consumer group. Memberi waktu lebih banyak consumer untuk join sebelum rebalance.
  # 0 = langsung rebalance (ok untuk dev/test).
  group.initial.rebalance.delay.ms: 0

  # ----------------------------------------------------------------
  # Auto Topic Creation
  # ----------------------------------------------------------------

  # auto.create.topics.enable: apakah broker otomatis membuat topic
  # saat producer/consumer pertama kali menggunakannya.
  # false = topic harus dibuat eksplisit (lebih aman untuk production).
  auto.create.topics.enable: true

# ============================================================
# Schema Registry Configuration
# ============================================================
schema_registry_custom_properties:
  # Listener Schema Registry (HTTP karena ssl_enabled=false)
  listeners: "http://0.0.0.0:8081"

  # schema.compatibility.level: aturan default kompatibilitas schema.
  # Opsi: BACKWARD, FORWARD, FULL, NONE, BACKWARD_TRANSITIVE, FORWARD_TRANSITIVE, FULL_TRANSITIVE
  # BACKWARD (default): consumer baru bisa membaca data lama dan baru.
  schema.compatibility.level: "BACKWARD"

  # Ukuran cache schema di memory Schema Registry
  schema.registry.resource.extension.version: "latest"

  # Aktifkan auto-registration schema oleh producer (default: true)
  # false = schema harus didaftarkan manual sebelum digunakan
  auto.register.schemas: true

# ============================================================
# Kafka Connect Configuration
# ============================================================
kafka_connect_custom_properties:
  # ----------------------------------------------------------------
  # Connector Plugin Settings
  # ----------------------------------------------------------------

  # plugin.path: direktori tempat Kafka Connect mencari connector JAR.
  # Tambahkan path jika ada custom connector.
  plugin.path: "/usr/share/java,/usr/share/confluent-hub-components,/home/cpadmin/confluent-ansible/plugins/connectors"

  # ----------------------------------------------------------------
  # Internal Topics (untuk state management Connect cluster)
  # ----------------------------------------------------------------

  # config.storage.topic: topic untuk menyimpan konfigurasi connector.
  # Gunakan replication factor = jumlah broker.
  config.storage.replication.factor: 3

  # offset.storage.topic: topic untuk menyimpan offset connector.
  offset.storage.replication.factor: 3

  # status.storage.topic: topic untuk menyimpan status task/connector.
  status.storage.replication.factor: 3

  # ----------------------------------------------------------------
  # Worker Settings
  # ----------------------------------------------------------------

  # key.converter: serializer untuk key message dari/ke Connect.
  key.converter: "org.apache.kafka.connect.storage.StringConverter"

  # value.converter: serializer untuk value message. Bisa dioverride per-connector.
  value.converter: "org.apache.kafka.connect.json.JsonConverter"

  # Sertakan schema dalam JSON value (berlaku jika menggunakan JsonConverter)
  value.converter.schemas.enable: "true"

  # consumer.max.poll.records: jumlah record max yang diambil per poll
  # untuk sink connector. Naikkan untuk throughput, turunkan untuk latency.
  consumer.max.poll.records: 500

# ============================================================
# ksqlDB Configuration
# ============================================================
ksql_custom_properties:
  # Nama aplikasi ksqlDB — digunakan sebagai prefix consumer group
  # dan nama internal topic. Harus unik per ksqlDB cluster.
  ksql.service.id: "confluent_ksql_"

  # Jumlah thread untuk pemrosesan stream. Lebih banyak = throughput lebih tinggi,
  # tapi lebih banyak resource. Rekomendasi: sama dengan jumlah partisi / jumlah node.
  ksql.streams.num.stream.threads: 4

  # ksql.streams.replication.factor: replikasi untuk internal topic ksqlDB
  # (changelog, repartition topics).
  ksql.streams.replication.factor: 3

  # Aktifkan mode headless (tanpa REST API) — false = interactive mode
  ksql.headless: false

  # Batas waktu (ms) untuk query ksqlDB via REST API
  ksql.server.query.pull.enable.offset.lag.prop: true

  # Maksimum baris yang dikembalikan oleh pull query (SELECT dari table)
  ksql.query.pull.max.hourly.bandwidth.megabytes: 2147483647

  # Aktifkan akses ke topic yang tidak berformat Avro
  ksql.schema.registry.url: "http://cp-node1:8081"

# ============================================================
# Kafka REST Proxy Configuration
# ============================================================
kafka_rest_custom_properties:
  # URL Schema Registry untuk serialisasi Avro via REST
  schema.registry.url: "http://cp-node1:8081"

  # Ukuran maksimum request (bytes) — default 64KB, naikkan untuk pesan besar
  max.request.size: 67108864

  # consumer.request.timeout.ms: timeout saat consumer menunggu pesan baru
  consumer.request.timeout.ms: 1000

  # consumer.instance.timeout.ms: waktu sebelum consumer instance dihapus
  # jika tidak ada aktivitas (idle timeout).
  consumer.instance.timeout.ms: 300000

  # consumer.threads: jumlah thread untuk menangani consumer request
  consumer.threads: 1

  # Aktifkan CORS untuk akses dari browser (opsional, berguna untuk development)
  access.control.allow.methods: "GET,POST,PUT,DELETE,OPTIONS,HEAD"
  access.control.allow.origin: "*"

# ============================================================
# Control Center Configuration
# ============================================================
control_center_custom_properties:
  # Replication factor untuk internal topic Control Center
  confluent.controlcenter.internal.topics.replication: 3

  # Retention untuk data monitoring (ms) — 7 hari
  confluent.controlcenter.data.collection.maxage.ms: 604800000

  # Jumlah partisi untuk internal topic Control Center
  confluent.controlcenter.internal.topics.partitions: 6

  # Interval (ms) monitoring metrics collection
  confluent.metrics.topic.replication: 3

  # Stream monitoring settings
  confluent.controlcenter.streams.num.stream.threads: 4

  # Ukuran rocksdb state store untuk stream processing
  confluent.controlcenter.streams.cache.max.bytes.buffering: 104857600

  # Nonaktifkan usage reporting ke Confluent (opsional)
  confluent.controlcenter.usage.data.collection.enable: false
EOF
```

### Penjelasan Konfigurasi Penting Per Komponen

**ZooKeeper — Parameter Kritis:**

| Property | Default | Nilai Kita | Alasan |
|----------|---------|------------|--------|
| `tickTime` | 2000 | 2000 | Unit waktu 2 detik, cukup responsif untuk lab |
| `initLimit` | 10 | 10 | 20 detik untuk sync saat startup — aman untuk lab |
| `syncLimit` | 5 | 5 | 10 detik timeout sync — cukup untuk jaringan lokal |
| `autopurge.snapRetainCount` | 3 | 3 | Simpan 3 snapshot = recovery point |
| `autopurge.purgeInterval` | 0 (off) | 1 | Bersihkan setiap jam — cegah disk penuh |

**Kafka Broker — Parameter Kritis:**

| Property | Default | Nilai Kita | Alasan |
|----------|---------|------------|--------|
| `default.replication.factor` | 1 | 3 | High availability — tahan 1 node failure |
| `min.insync.replicas` | 1 | 2 | Durability — data tidak hilang jika 1 broker mati |
| `num.partitions` | 1 | 3 | Parallelism sejak awal |
| `log.retention.hours` | 168 | 168 | 7 hari — standar untuk lab |
| `auto.create.topics.enable` | true | true | Mudah untuk development/testing |

---

## Step 4: Buat Playbook Utama

Playbook adalah file YAML yang menginstruksikan Ansible apa yang harus dilakukan di node mana, dan dalam urutan apa.

```bash
cat > ~/confluent-ansible/confluent.yml << 'EOF'
---
# ============================================================
# Main Playbook: Deploy Confluent Platform Full Stack
# Task 2 — No Security (Plaintext)
# Urutan deployment PENTING: ZooKeeper harus selesai sebelum
# Kafka Broker, dan Kafka Broker harus selesai sebelum
# semua komponen lain.
# ============================================================

# ============================================================
# Phase 0: Load Variables
# Membaca semua konfigurasi dari vars/platform.yml
# dan menyediakan ke semua hosts sebelum deployment dimulai.
# ============================================================
- name: Load platform variables
  hosts: all
  gather_facts: false
  tasks:
    - name: Include platform variables
      ansible.builtin.include_vars:
        file: "{{ playbook_dir }}/vars/platform.yml"

# ============================================================
# Phase 1: Deploy ZooKeeper
# ZooKeeper harus running SEBELUM Kafka Broker bisa distart.
# Kafka Broker membutuhkan ZooKeeper untuk:
#   - Menyimpan metadata broker
#   - Leader election controller
#   - Konfigurasi ACL (meski kita tidak pakai security)
# ============================================================
- name: ZooKeeper
  import_playbook: confluent.platform.zookeeper

# ============================================================
# Phase 2: Deploy Kafka Broker
# Kafka Broker harus running SEBELUM semua komponen lain karena:
#   - Schema Registry menggunakan Kafka topic untuk storage
#   - Kafka Connect menggunakan Kafka topic untuk state
#   - ksqlDB menggunakan Kafka topic untuk changelog
#   - Control Center menggunakan Kafka topic untuk metrics
# ============================================================
- name: Kafka Broker
  import_playbook: confluent.platform.kafka_broker

# ============================================================
# Phase 3: Deploy Schema Registry
# Schema Registry bisa distart setelah Kafka Broker running.
# Schema Registry:
#   - Menyimpan schema di Kafka topic: _schemas
#   - Melayani request serialisasi/deserialisasi dari producer/consumer
#   - Dibutuhkan oleh ksqlDB (untuk Avro support) dan REST Proxy
# ============================================================
- name: Schema Registry
  import_playbook: confluent.platform.schema_registry

# ============================================================
# Phase 4: Deploy Kafka Connect
# Connect Worker bisa distart setelah Kafka Broker running.
# Kafka Connect:
#   - Menyimpan state di 3 internal Kafka topic:
#     * connect-configs   (konfigurasi connector)
#     * connect-offsets   (posisi offset source connector)
#     * connect-status    (status task dan connector)
#   - Berjalan dalam distributed mode untuk HA
# ============================================================
- name: Kafka Connect
  import_playbook: confluent.platform.kafka_connect

# ============================================================
# Phase 5: Deploy ksqlDB
# ksqlDB distart setelah Kafka dan Schema Registry running.
# ksqlDB:
#   - Membutuhkan Kafka untuk stream processing
#   - Opsional terintegrasi dengan Schema Registry (Avro)
#   - Menyimpan persistent query state di Kafka changelog topics
# ============================================================
- name: ksqlDB
  import_playbook: confluent.platform.ksql

# ============================================================
# Phase 6: Deploy Kafka REST Proxy
# REST Proxy distart setelah Kafka dan Schema Registry running.
# REST Proxy:
#   - Menyediakan HTTP REST API untuk Kafka
#   - Terintegrasi dengan Schema Registry untuk Avro
#   - Berguna untuk klien yang tidak bisa pakai Kafka protocol
# ============================================================
- name: Kafka REST Proxy
  import_playbook: confluent.platform.kafka_rest

# ============================================================
# Phase 7: Deploy Control Center
# Control Center distart TERAKHIR setelah semua komponen lain.
# Control Center:
#   - Membutuhkan Kafka untuk internal metrics storage
#   - Memonitor semua komponen: ZooKeeper, Broker, SR, Connect, ksqlDB
#   - Diinstall di cp-ansible (bukan di data nodes)
# ============================================================
- name: Control Center
  import_playbook: confluent.platform.control_center
EOF
```

**Mengapa Urutan Deployment Penting?**

Confluent Platform memiliki dependency chain yang harus diikuti:

```
ZooKeeper
    ↓
Kafka Broker (butuh ZooKeeper untuk metadata)
    ↓
Schema Registry (butuh Kafka untuk topic _schemas)
Kafka Connect   (butuh Kafka untuk connect-configs, dll)
ksqlDB          (butuh Kafka + Schema Registry opsional)
REST Proxy      (butuh Kafka + Schema Registry)
Control Center  (butuh semua komponen di atas)
```

**Verifikasi Struktur Project:**

```bash
tree ~/confluent-ansible/
```

Output yang diharapkan:
```
/home/cpadmin/confluent-ansible/
├── certs/
├── confluent.yml
├── inventory/
│   └── hosts.yml
├── plugins/
│   └── connectors/
└── vars/
    └── platform.yml
```

---

## Step 5: Verifikasi Syntax dan Konektivitas

### 5a. Cek Syntax Playbook

```bash
cd ~/confluent-ansible

ansible-playbook -i inventory/hosts.yml confluent.yml --syntax-check
```
<img width="1341" height="468" alt="image" src="https://github.com/user-attachments/assets/0c81dae3-412a-4813-8a52-148663017ccd" />

Output yang diharapkan:
```
playbook: confluent.yml
  play #1 (all): Load platform variables
  play #2 (zookeeper): ZooKeeper
  play #3 (kafka_broker): Kafka Broker
  play #4 (schema_registry): Schema Registry
  play #5 (kafka_connect): Kafka Connect
  play #6 (ksql): ksqlDB
  play #7 (kafka_rest): Kafka REST Proxy
  play #8 (control_center): Control Center
```
> **WARNING** tentang `zookeeper_parallel`, `zookeeper_serial`, dll — ini normal, hanya informasi bahwa Confluent Ansible collection mengharapkan subgroup tersebut tapi tidak wajib dibuat manual.
> **Jika ada error syntax**, Ansible akan menampilkan baris dan file yang bermasalah. Perbaiki sebelum lanjut.

### 5b. Test Konektivitas SSH ke Semua Node

```bash
ansible -i inventory/hosts.yml all -m ping
```
<img width="1121" height="701" alt="image" src="https://github.com/user-attachments/assets/4e566929-f635-41e6-ba92-77adbc4d67b9" />

Output yang diharapkan:
```
cp-ansible | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
cp-node1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
cp-node2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
cp-node3 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

✅ **Semua `SUCCESS` → konektivitas OK.**

### 5c. Verifikasi Sudah Bisa sudo

```bash
# Pastikan ansible_become=true berfungsi — test dengan perintah yang butuh root
ansible -i inventory/hosts.yml all -m command -a "whoami" --become
```
<img width="1293" height="258" alt="image" src="https://github.com/user-attachments/assets/3d56e22a-ee3b-45e6-bccb-9840a7951100" />

Output yang diharapkan:
```
cp-node1 | CHANGED | rc=0 >>
root
cp-node2 | CHANGED | rc=0 >>
root
cp-node3 | CHANGED | rc=0 >>
root
cp-ansible | CHANGED | rc=0 >>
root
```

### 5d. Verifikasi Variabel Efektif

```bash
# Lihat variabel yang akan dipakai untuk cp-node1
ansible -i inventory/hosts.yml cp-node1 -m debug \
  -a "var=confluent_package_version,ssl_enabled,sasl_protocol" \
  -e "@vars/platform.yml"
```

<img width="1105" height="175" alt="image" src="https://github.com/user-attachments/assets/8a195b82-04f5-4e28-a529-200ff099363c" />

Output yang diharapkan:
```json
{
    "confluent_package_version": "7.9.5",
    "ssl_enabled": false,
    "sasl_protocol": "none"
}
```

### 5e. Cek Disk Space di Semua Node

```bash
# Pastikan ada cukup disk space (minimal 10GB per node untuk full install)
ansible -i inventory/hosts.yml all -m shell \
  -a "df -h / | tail -1"
```

<img width="1048" height="295" alt="image" src="https://github.com/user-attachments/assets/e26c53c9-d44e-49f2-b109-909bf14d1a44" />

---
