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

## Step 6: Jalankan Deployment

> ⚠️ **Pastikan semua verifikasi di Step 5 berhasil sebelum menjalankan ini.**
> Proses deployment full stack membutuhkan **20–45 menit** tergantung kecepatan internet untuk download package.

```bash
cd ~/confluent-ansible

ansible-playbook -i inventory/hosts.yml confluent.yml
```

**Jalankan dengan verbose jika ada masalah:**

```bash
# -v: task info
# -vv: + variabel
# -vvv: + SSH detail
ansible-playbook -i inventory/hosts.yml confluent.yml -vv 2>&1 | tee ~/deployment.log
```

**Jalankan hanya komponen tertentu (via tags):**

```bash
# Hanya deploy ZooKeeper dan Kafka
ansible-playbook -i inventory/hosts.yml confluent.yml \
  --tags zookeeper,kafka_broker

# Hanya deploy Schema Registry
ansible-playbook -i inventory/hosts.yml confluent.yml \
  --tags schema_registry

# Hanya deploy Control Center
ansible-playbook -i inventory/hosts.yml confluent.yml \
  --tags control_center
```

### Proses yang Terjadi Selama Deployment

Ansible akan menjalankan task secara berurutan untuk setiap komponen:

**Fase ZooKeeper (di cp-node1, cp-node2, cp-node3):**
```
TASK [Install ZooKeeper Package] ............... changed
TASK [Create ZooKeeper Data Directory] ......... changed
TASK [Create ZooKeeper myid File] .............. changed
TASK [Write ZooKeeper Configuration] ........... changed
TASK [Enable ZooKeeper Service] ................ changed
TASK [Start ZooKeeper Service] ................. changed
TASK [Wait for ZooKeeper Port 2181] ............ ok
```

**Fase Kafka Broker (di cp-node1, cp-node2, cp-node3):**
```
TASK [Install Kafka Broker Package] ............ changed
TASK [Write server.properties] ................. changed
TASK [Enable Kafka Service] .................... changed
TASK [Start Kafka Service] ..................... changed
TASK [Wait for Kafka Port 9092] ................ ok
TASK [Wait for Kafka Broker to be Registered] .. ok
```

**Fase Schema Registry (di cp-node1, cp-node2, cp-node3):**
```
TASK [Install Schema Registry Package] ......... changed
TASK [Write schema-registry.properties] ........ changed
TASK [Start Schema Registry Service] ........... changed
TASK [Wait for Schema Registry Port 8081] ...... ok
```

**Fase Kafka Connect (di cp-node1, cp-node2, cp-node3):**
```
TASK [Install Kafka Connect Package] ........... changed
TASK [Write connect-distributed.properties] .... changed
TASK [Create Internal Topics for Connect] ...... changed
TASK [Start Kafka Connect Service] ............. changed
TASK [Wait for Kafka Connect Port 8083] ........ ok
```

**Fase ksqlDB (di cp-node1, cp-node2, cp-node3):**
```
TASK [Install ksqlDB Package] .................. changed
TASK [Write ksql-server.properties] ............ changed
TASK [Start ksqlDB Service] .................... changed
TASK [Wait for ksqlDB Port 8088] ............... ok
```

**Fase REST Proxy (di cp-node1, cp-node2, cp-node3):**
```
TASK [Install REST Proxy Package] .............. changed
TASK [Write kafka-rest.properties] ............. changed
TASK [Start REST Proxy Service] ................ changed
TASK [Wait for REST Proxy Port 8082] ........... ok
```

**Fase Control Center (di cp-ansible):**
```
TASK [Install Control Center Package] .......... changed
TASK [Write control-center.properties] ......... changed
TASK [Start Control Center Service] ............ changed
TASK [Wait for Control Center Port 9021] ....... ok
```

### Output Deployment yang Normal

```
PLAY RECAP *****************************************************
cp-ansible  : ok=45   changed=30   unreachable=0   failed=0
cp-node1    : ok=180  changed=120  unreachable=0   failed=0
cp-node2    : ok=180  changed=120  unreachable=0   failed=0
cp-node3    : ok=180  changed=120  unreachable=0   failed=0
```
<img width="1526" height="193" alt="image" src="https://github.com/user-attachments/assets/1e5ff65c-76a1-4871-8c8e-7868f99e4714" />


✅ **Deployment berhasil jika `failed=0` dan `unreachable=0` untuk semua node.**

---
## Pengujian dan Verifikasi Deployment

---

### Test 1: Verifikasi Status Semua Service Sekaligus

Jalankan dari `cp-ansible` menggunakan script berikut:

```bash
cd ~/confluent-ansible

cat > /tmp/check_services.sh << 'EOF'
#!/bin/bash
echo "========================================"
echo " Confluent Platform Service Status Check"
echo "========================================"

SERVICES=(
  "confluent-zookeeper"
  "confluent-server"
  "confluent-schema-registry"
  "confluent-kafka-connect"
  "confluent-ksqldb"
  "confluent-kafka-rest"
)

for svc in "${SERVICES[@]}"; do
  STATUS=$(systemctl is-active $svc 2>/dev/null)
  if [ "$STATUS" = "active" ]; then
    echo "✅ $svc: RUNNING"
  else
    echo "❌ $svc: $STATUS"
  fi
done
EOF
```
```
# Jalankan script di semua data nodes
ansible -i inventory/hosts.yml zookeeper -m script -a /tmp/check_services.sh

# Cek Control Center di cp-ansible
ansible -i inventory/hosts.yml control_center \
  -m shell -a "systemctl is-active confluent-control-center"
```
<img width="1558" height="557" alt="image" src="https://github.com/user-attachments/assets/147cd107-9175-41a8-8b71-939ec0a98109" />

<img width="1509" height="121" alt="image" src="https://github.com/user-attachments/assets/95e6c1ba-cfbc-4682-919c-bc48d7274105" />

> **⚠️ Catatan Nama Service:** Nama service Kafka Broker di Confluent Platform adalah `confluent-server`, **bukan** `confluent-kafka`. Pastikan script menggunakan nama yang benar.

Output yang diharapkan di setiap data node:

```
✅ confluent-zookeeper: RUNNING
✅ confluent-server: RUNNING
✅ confluent-schema-registry: RUNNING
✅ confluent-kafka-connect: RUNNING
✅ confluent-ksqldb: RUNNING
✅ confluent-kafka-rest: RUNNING
```

---

### Test 2: Verifikasi Port Listening di Semua Node

```bash
# Cek semua port Confluent di semua data nodes
ansible -i ~/confluent-ansible/inventory/hosts.yml zookeeper \
  -m shell -a "ss -tlnp | grep -E '2181|2888|3888|9092|8081|8082|8083|8088' | awk '{print \$4, \$6}'"
```

Output yang diharapkan per node (port 2888 hanya muncul di node ZooKeeper leader):

```
*:3888  users:(("java",...))   # ZooKeeper leader election
*:2181  users:(("java",...))   # ZooKeeper client
*:9092  users:(("java",...))   # Kafka Broker
*:8081  users:(("java",...))   # Schema Registry
*:8082  users:(("java",...))   # REST Proxy
*:8083  users:(("java",...))   # Kafka Connect
*:8088  users:(("java",...))   # ksqlDB
```

> **Catatan:** Port `2888` (ZooKeeper peer) hanya akan muncul di node yang menjadi ZooKeeper **leader**. Node follower tidak membuka port ini secara aktif.

```bash
# Cek port Control Center di cp-ansible
ansible -i ~/confluent-ansible/inventory/hosts.yml control_center \
  -m shell -a "ss -tlnp | grep 9021 | awk '{print \$4, \$6}'"
```

---

### Test 3: ZooKeeper Health Check

> **⚠️ Catatan:** Confluent Platform menonaktifkan ZooKeeper 4-letter words (`ruok`, `stat`, `mntr`) secara default karena alasan keamanan. Gunakan `srvr` sebagai pengganti atau gunakan `zookeeper-shell`.

```bash
# Cek ZooKeeper menggunakan srvr (tersedia meski 4lw lain dinonaktifkan)
for node in cp-node1 cp-node2 cp-node3; do
  echo "=== $node ==="
  echo "srvr" | nc $node 2181 | grep -E "Zookeeper version|Mode|Node count"
  echo ""
done
```

Output yang diharapkan:

<img width="1039" height="374" alt="image" src="https://github.com/user-attachments/assets/59471948-204f-4d74-adb3-0785a5e6e6af" />

```
=== cp-node1 ===
Zookeeper version: 3.8.4-...
Mode: follower
Node count: 5

=== cp-node2 ===
Mode: leader
Node count: 5

=== cp-node3 ===
Mode: follower
Node count: 5
```

```bash
# Verifikasi quorum aktif menggunakan zookeeper-shell
/usr/bin/zookeeper-shell cp-node1:2181 ls /brokers/ids
```

Output yang diharapkan:

<img width="1018" height="167" alt="image" src="https://github.com/user-attachments/assets/c7b1f714-d305-4c0a-b8e3-653fe028a013" />

```
[1, 2, 3]
```

✅ **Ketiga broker terdaftar = ZooKeeper ensemble dan Kafka cluster sehat.**

---

### Test 4: Kafka Broker Health Check

```bash
# Cek daftar broker yang terdaftar di ZooKeeper
/usr/bin/zookeeper-shell cp-node1:2181 ls /brokers/ids
# Output: [1, 2, 3]
```

```bash
# Cek detail setiap broker
for id in 1 2 3; do
  echo "=== Broker $id ==="
  /usr/bin/zookeeper-shell cp-node1:2181 get /brokers/ids/$id
  echo ""
done
```

Output contoh untuk Broker 1:

<img width="1870" height="644" alt="image" src="https://github.com/user-attachments/assets/1e16359a-d709-42ea-951b-b165e4336d9d" />

```json
{
  "listener_security_protocol_map": {"INTERNAL":"PLAINTEXT","BROKER":"PLAINTEXT"},
  "endpoints": ["INTERNAL://cp-node1:9092","BROKER://cp-node1:9091"],
  "host": "cp-node1",
  "port": 9092,
  "version": 5
}
```

```bash
# Cek Kafka controller (broker yang menjadi cluster controller)
/usr/bin/zookeeper-shell cp-node1:2181 get /controller
# Output: {"version":2,"brokerid":2,...}
```

```bash
# Cek list topic dari salah satu data node
ssh cpadmin@cp-node1 "kafka-topics --bootstrap-server cp-node1:9092 --list"
```
<img width="963" height="177" alt="image" src="https://github.com/user-attachments/assets/d4ae4b1c-8632-48f1-b50d-97fa9f5422b0" />


> **⚠️ Catatan:** `kafka-broker-api-versions` **tidak bisa dijalankan dari cp-ansible** karena cp-ansible tidak berada di dalam network internal Kafka. Jalankan Kafka CLI tools dari salah satu data node (cp-node1/2/3) melalui SSH.

```bash
# Cara yang benar: jalankan dari data node
ssh cpadmin@cp-node1 "kafka-broker-api-versions --bootstrap-server cp-node1:9092 | head -20"
```

---

### Test 5: Schema Registry Health Check

```bash
# Cek status Schema Registry
curl -s http://cp-node1:8081/ | python3 -m json.tool
# Output: {}  (response kosong adalah normal untuk endpoint root)
```

```bash
# Cek konfigurasi compatibility mode
curl -s http://cp-node1:8081/config | python3 -m json.tool
```

Output:

<img width="1005" height="127" alt="image" src="https://github.com/user-attachments/assets/89bd542f-c98f-4482-9a16-c8aef3c22842" />

```json
{
    "compatibilityLevel": "BACKWARD"
}
```

```bash
# Daftarkan schema string sederhana
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "{\"type\": \"string\"}"}' \
  http://cp-node1:8081/subjects/test-string-value/versions
# Output: {"id":1}
```
<img width="1555" height="134" alt="image" src="https://github.com/user-attachments/assets/0dd4b564-8ac2-40be-9cc1-dc0095e4ec6b" />


> **Catatan:** Request pertama ke Schema Registry mungkin mengalami timeout singkat (`Register operation timed out`) saat Schema Registry pertama kali aktif. Ulangi request — biasanya berhasil di percobaan kedua.

```bash
# Daftarkan schema Avro kompleks (User record)
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{
    "schema": "{\"type\":\"record\",\"name\":\"User\",\"fields\":[{\"name\":\"id\",\"type\":\"int\"},{\"name\":\"name\",\"type\":\"string\"},{\"name\":\"email\",\"type\":\"string\"}]}"
  }' \
  http://cp-node1:8081/subjects/user-value/versions
# Output: {"id":2}
```

```bash
# Lihat semua subject yang terdaftar
curl -s http://cp-node1:8081/subjects
# Output: ["test-string-value","user-value"]

# Lihat detail schema
curl -s http://cp-node1:8081/subjects/user-value/versions/1 | python3 -m json.tool
```

```bash
# Test backward compatibility — tambah field opsional (harus kompatibel)
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{
    "schema": "{\"type\":\"record\",\"name\":\"User\",\"fields\":[{\"name\":\"id\",\"type\":\"int\"},{\"name\":\"name\",\"type\":\"string\"},{\"name\":\"email\",\"type\":\"string\"},{\"name\":\"age\",\"type\":[\"null\",\"int\"],\"default\":null}]}"
  }' \
  http://cp-node1:8081/compatibility/subjects/user-value/versions/latest
# Output: {"is_compatible":true}
```
<img width="1563" height="171" alt="image" src="https://github.com/user-attachments/assets/7dc37e81-3998-4fee-9440-d32719d7e787" />

<img width="1559" height="424" alt="image" src="https://github.com/user-attachments/assets/d83bf013-a957-404a-8e84-9a5caf1581ac" />

---

### Test 6: Kafka Connect Health Check

```bash
# Cek status cluster Kafka Connect
curl -s http://cp-node1:8083/ | python3 -m json.tool
```

Output:

<img width="991" height="119" alt="image" src="https://github.com/user-attachments/assets/6a1edc95-c14c-49af-a16e-444093a0983c" />

```json
{
    "version": "7.9.5-ce",
    "commit": "21fdf52c487846be",
    "kafka_cluster_id": "S8oRfvb0TfOTv6Txhg2g0g"
}
```

```bash
# Lihat semua connector plugins yang tersedia
curl -s http://cp-node1:8083/connector-plugins | python3 -m json.tool
```

Output (default Confluent Platform tanpa tambahan plugin):

<img width="1164" height="488" alt="image" src="https://github.com/user-attachments/assets/1c09be40-40f0-449e-bf29-811f82191a78" />

```json
[
    {
        "class": "io.confluent.connect.replicator.ReplicatorSourceConnector",
        "type": "source",
        "version": "7.9.5"
    },
    {
        "class": "org.apache.kafka.connect.mirror.MirrorCheckpointConnector",
        "type": "source",
        "version": "7.9.5-ce"
    },
    {
        "class": "org.apache.kafka.connect.mirror.MirrorHeartbeatConnector",
        "type": "source",
        "version": "7.9.5-ce"
    },
    {
        "class": "org.apache.kafka.connect.mirror.MirrorSourceConnector",
        "type": "source",
        "version": "7.9.5-ce"
    }
]
```

> **Catatan:** `FileStreamSource` dan `FileStreamSink` **tidak tersedia** di Confluent Platform karena tidak disertakan dalam package `confluent-server`. Connector tersebut hanya ada di Apache Kafka open-source. Untuk pengujian connector, gunakan `MirrorSourceConnector` atau install connector tambahan via `confluent-hub`.

```bash
# Lihat semua connector yang sedang berjalan
curl -s http://cp-node1:8083/connectors
# Output: []
```

**Install connector tambahan (opsional) via confluent-hub:**

```bash
# Contoh install JDBC Connector
ssh cpadmin@cp-node1
confluent-hub install confluentinc/kafka-connect-jdbc:latest --no-prompt \
  --component-dir /usr/share/confluent-hub-components

# Restart Connect agar connector baru terdeteksi
sudo systemctl restart confluent-kafka-connect
```

---

### Test 7: ksqlDB Health Check

```bash
# Cek status ksqlDB server
curl -s http://cp-node1:8088/info | python3 -m json.tool
```
<img width="994" height="267" alt="image" src="https://github.com/user-attachments/assets/e0a81dfd-7d3f-4543-85ff-2eacf2581a2e" />

Output:

```json
{
    "KsqlServerInfo": {
        "version": "7.9.5",
        "kafkaClusterId": "S8oRfvb0TfOTv6Txhg2g0g",
        "ksqlServiceId": "confluent_ksql_",
        "serverStatus": "RUNNING"
    }
}
```

> **Catatan:** CLI `ksql` **tidak tersedia di cp-ansible**. Jalankan ksqlDB CLI dari salah satu data node melalui SSH.

```bash
# Jalankan ksqlDB CLI dari data node
ssh cpadmin@cp-node1 "ksql http://cp-node1:8088"
```

Di dalam ksqlDB CLI:

```sql
-- Lihat semua topic Kafka yang tersedia
SHOW TOPICS;

-- Lihat semua stream (kosong di awal)
SHOW STREAMS;

-- Lihat semua table (kosong di awal)
SHOW TABLES;
```
<img width="1066" height="710" alt="image" src="https://github.com/user-attachments/assets/50aa4821-af8a-4c1c-92c6-e270b6f81699" />

```bash
# Alternatif: query ksqlDB via REST API tanpa CLI
curl -X POST http://cp-node1:8088/ksql \
  -H "Content-Type: application/vnd.ksql.v1+json; charset=utf-8" \
  --data '{"ksql": "SHOW TOPICS;", "streamsProperties": {}}' | python3 -m json.tool
```
<img width="872" height="719" alt="image" src="https://github.com/user-attachments/assets/37c54aca-2be1-497e-a9ef-28e72b00ab2f" />

---

### Test 8: Kafka REST Proxy Health Check

```bash
# Cek status REST Proxy
curl -s http://cp-node1:8082/ | python3 -m json.tool
# Output: {}  (response kosong pada endpoint root adalah normal)
```

```bash
# Lihat semua topic via REST API
curl -s http://cp-node1:8082/topics | python3 -m json.tool
```

Output (akan berisi banyak internal topic Confluent):

<img width="1116" height="705" alt="image" src="https://github.com/user-attachments/assets/636060aa-8d5c-4a83-a297-ce9959fddb98" />

```json
[
    "_schemas",
    "_confluent-metrics",
    "_confluent-monitoring",
    "connect-cluster-configs",
    "connect-cluster-offsets",
    "connect-cluster-status",
    "_confluent-ksql-confluent_ksql__command_topic",
    ...
]
```

**Produce pesan JSON via REST API:**

```bash
curl -X POST \
  -H "Content-Type: application/vnd.kafka.json.v2+json" \
  -H "Accept: application/vnd.kafka.v2+json" \
  --data '{
    "records": [
      {"value": {"message": "Hello from REST!", "source": "rest-proxy"}},
      {"value": {"message": "REST Proxy works!", "source": "rest-proxy"}}
    ]
  }' \
  http://cp-node1:8082/topics/test-topic
```

Output sukses:
<img width="1864" height="171" alt="image" src="https://github.com/user-attachments/assets/11464098-f7b8-4ed2-b055-b44696fd0a28" />

```json
{
    "offsets": [
        {"partition": 0, "offset": 0, "error_code": null, "error": null},
        {"partition": 1, "offset": 0, "error_code": null, "error": null}
    ],
    "key_schema_id": null,
    "value_schema_id": null
}
```

**Produce pesan Avro via REST API + Schema Registry:**

```bash
curl -X POST \
  -H "Content-Type: application/vnd.kafka.avro.v2+json" \
  -H "Accept: application/vnd.kafka.v2+json" \
  --data '{
    "value_schema": "{\"type\":\"record\",\"name\":\"Payment\",\"fields\":[{\"name\":\"amount\",\"type\":\"double\"},{\"name\":\"currency\",\"type\":\"string\"}]}",
    "records": [
      {"value": {"amount": 100.50, "currency": "USD"}},
      {"value": {"amount": 250.00, "currency": "EUR"}}
    ]
  }' \
  http://cp-node1:8082/topics/payments-avro
```

Output sukses (schema otomatis didaftarkan ke Schema Registry):

<img width="1651" height="275" alt="image" src="https://github.com/user-attachments/assets/c08d2a57-9b69-4b14-bf5a-2429abd082ed" />

```json
{
    "offsets": [
        {"partition": 2, "offset": 0, "error_code": null, "error": null},
        {"partition": 2, "offset": 1, "error_code": null, "error": null}
    ],
    "key_schema_id": null,
    "value_schema_id": 3
}
```

✅ **`value_schema_id: 3` menunjukkan schema Payment berhasil terdaftar ke Schema Registry.**

**Consume pesan via REST API:**

```bash
# Langkah 1: Buat consumer instance
curl -X POST \
  -H "Content-Type: application/vnd.kafka.v2+json" \
  --data '{"name": "rest-consumer-1", "format": "json", "auto.offset.reset": "earliest"}' \
  http://cp-node1:8082/consumers/rest-consumer-group

# Langkah 2: Subscribe ke topic
curl -X POST \
  -H "Content-Type: application/vnd.kafka.v2+json" \
  --data '{"topics": ["test-topic"]}' \
  http://cp-node1:8082/consumers/rest-consumer-group/instances/rest-consumer-1/subscription

# Langkah 3: Consume records
curl -X GET \
  -H "Accept: application/vnd.kafka.json.v2+json" \
  http://cp-node1:8082/consumers/rest-consumer-group/instances/rest-consumer-1/records

# Langkah 4: Hapus consumer instance
curl -X DELETE \
  -H "Content-Type: application/vnd.kafka.v2+json" \
  http://cp-node1:8082/consumers/rest-consumer-group/instances/rest-consumer-1
```
<img width="1692" height="407" alt="image" src="https://github.com/user-attachments/assets/81ff6740-3f8f-4609-a20d-fe78bc55cd4e" />


---

### Test 9: Control Center Health Check

Control Center dapat diakses melalui browser:

```
http://192.168.56.10:9021
```
<img width="1411" height="933" alt="image" src="https://github.com/user-attachments/assets/088ee663-617c-4520-8fee-524f713e66f6" />

```bash
# Cek HTTP response dari cp-ansible
curl -s -o /dev/null -w "%{http_code}" http://192.168.56.10:9021
# Output: 200
```

**Dashboard Control Center menampilkan:**

| Informasi | Nilai |
|-----------|-------|
| Brokers | 3 |
| Partitions | 406+ |
| Topics | 59+ |
| ksqlDB clusters | 1 |
| Connect clusters | 1 |

> **Catatan:** Control Center mungkin menampilkan **"1 Unhealthy cluster"** saat pertama kali diakses. Ini normal — Control Center membutuhkan beberapa menit untuk mengumpulkan metrics dari semua komponen. Status akan berubah menjadi "Healthy" setelah data monitoring terkumpul.

**Fitur yang dapat dicek di browser:**

| URL | Fitur |
|-----|-------|
| `http://192.168.56.10:9021` | Dashboard utama |
| `http://192.168.56.10:9021/#/clusters` | Daftar cluster Kafka |
| `http://192.168.56.10:9021/#/topics` | Topic explorer |
| `http://192.168.56.10:9021/#/connect` | Kafka Connect management |
| `http://192.168.56.10:9021/#/ksql` | ksqlDB query editor |
| `http://192.168.56.10:9021/#/consumers` | Consumer group monitoring |

---

### Test 10: End-to-End Pipeline Test

Test ini membuktikan seluruh stack berfungsi bersama: data masuk via REST Proxy → diproses ksqlDB → dikonsumsi via kafka-console-consumer.

**Langkah 1:** Buat topic:

```bash
ssh cpadmin@cp-node1 "kafka-topics --bootstrap-server cp-node1:9092 \
  --create --topic orders-raw \
  --partitions 3 --replication-factor 3"

ssh cpadmin@cp-node1 "kafka-topics --bootstrap-server cp-node1:9092 \
  --create --topic orders-premium \
  --partitions 3 --replication-factor 3"
```

**Langkah 2:** Buat stream ksqlDB (dari cp-node1 via SSH):

```bash
ssh cpadmin@cp-node1 "ksql http://cp-node1:8088 <<'EOF'
CREATE STREAM orders_raw (
  order_id VARCHAR,
  customer VARCHAR,
  amount DOUBLE,
  status VARCHAR
) WITH (
  KAFKA_TOPIC = 'orders-raw',
  VALUE_FORMAT = 'JSON'
);

CREATE STREAM orders_premium AS
  SELECT order_id, customer, amount, status
  FROM orders_raw
  WHERE amount > 1000
  EMIT CHANGES;
EOF"
```

**Langkah 3:** Kirim data via REST Proxy (dari cp-ansible):

```bash
curl -X POST \
  -H "Content-Type: application/vnd.kafka.json.v2+json" \
  --data '{
    "records": [
      {"value": {"order_id": "ORD-001", "customer": "Alice", "amount": 500, "status": "pending"}},
      {"value": {"order_id": "ORD-002", "customer": "Bob", "amount": 1500, "status": "pending"}},
      {"value": {"order_id": "ORD-003", "customer": "Charlie", "amount": 250, "status": "pending"}},
      {"value": {"order_id": "ORD-004", "customer": "Diana", "amount": 2000, "status": "pending"}}
    ]
  }' \
  http://cp-node1:8082/topics/orders-raw
```

**Langkah 4:** Konsumsi hanya order premium (dari cp-node1):

```bash
ssh cpadmin@cp-node1 "kafka-console-consumer \
  --bootstrap-server cp-node1:9092 \
  --topic ORDERS_PREMIUM \
  --from-beginning \
  --timeout-ms 5000"
```

Output yang diharapkan:

```json
{"ORDER_ID":"ORD-002","CUSTOMER":"Bob","AMOUNT":1500.0,"STATUS":"pending"}
{"ORDER_ID":"ORD-004","CUSTOMER":"Diana","AMOUNT":2000.0,"STATUS":"pending"}
```

✅ **Hanya 2 record (amount > 1000) muncul — pipeline REST Proxy → Kafka → ksqlDB → Consumer berfungsi.**

---

### Test 11: Schema Evolution Test

```bash
# Schema v1: User dengan 2 field
curl -X POST \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "{\"type\":\"record\",\"name\":\"User\",\"fields\":[{\"name\":\"id\",\"type\":\"int\"},{\"name\":\"name\",\"type\":\"string\"}]}"}' \
  http://cp-node1:8081/subjects/users-value/versions
# Output: {"id":4}

# Schema v2: tambah field opsional (BACKWARD COMPATIBLE)
curl -X POST \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "{\"type\":\"record\",\"name\":\"User\",\"fields\":[{\"name\":\"id\",\"type\":\"int\"},{\"name\":\"name\",\"type\":\"string\"},{\"name\":\"email\",\"type\":[\"null\",\"string\"],\"default\":null}]}"}' \
  http://cp-node1:8081/subjects/users-value/versions
# Output: {"id":5} — berhasil karena backward compatible

# Schema v3: hapus field yang ada (BREAKING CHANGE — harus ditolak)
curl -X POST \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "{\"type\":\"record\",\"name\":\"User\",\"fields\":[{\"name\":\"id\",\"type\":\"int\"}]}"}' \
  http://cp-node1:8081/subjects/users-value/versions
# Output: HTTP 409 — Schema ditolak karena tidak backward compatible
```

---

### Test 12: Fault Tolerance — Node Down Test

**Langkah 1:** Hentikan semua service di cp-node3:

```bash
ssh cpadmin@cp-node3 "sudo systemctl stop \
  confluent-kafka-rest \
  confluent-ksqldb \
  confluent-kafka-connect \
  confluent-schema-registry \
  confluent-server \
  confluent-zookeeper"
```

**Langkah 2:** Verifikasi cluster masih beroperasi (quorum 2 dari 3 node):

```bash
# ZooKeeper masih quorum?
echo "srvr" | nc cp-node1 2181 | grep Mode

# Kafka masih bisa list topic?
ssh cpadmin@cp-node1 "kafka-topics --bootstrap-server cp-node1:9092 --list"

# Schema Registry masih merespons?
curl -s http://cp-node1:8081/subjects

# Produce pesan saat 1 node down
ssh cpadmin@cp-node1 "echo 'Message while cp-node3 is down' | \
  kafka-console-producer --bootstrap-server cp-node1:9092 --topic test-topic"
```

✅ **Semua operasi masih berfungsi karena quorum masih terpenuhi (2 dari 3 node aktif).**

**Langkah 3:** Hidupkan kembali cp-node3:

```bash
# ZooKeeper harus start pertama
ssh cpadmin@cp-node3 "sudo systemctl start confluent-zookeeper"
sleep 15  # Tunggu join ensemble

# Kafka broker
ssh cpadmin@cp-node3 "sudo systemctl start confluent-server"
sleep 30  # Tunggu rejoin dan replica sync

# Komponen lainnya
ssh cpadmin@cp-node3 "sudo systemctl start \
  confluent-schema-registry \
  confluent-kafka-connect \
  confluent-ksqldb \
  confluent-kafka-rest"
```

**Langkah 4:** Verifikasi cp-node3 rejoin:

```bash
# ZooKeeper ensemble kembali 3 node?
echo "srvr" | nc cp-node3 2181 | grep Mode

# Kafka broker IDs kembali [1,2,3]?
/usr/bin/zookeeper-shell cp-node1:2181 ls /brokers/ids
```

---

### Checklist Verifikasi Lengkap

```
DEPLOYMENT
[ ] ansible-playbook berhasil: failed=0, unreachable=0 untuk semua node
[ ] Semua 8 play selesai tanpa error

ZOOKEEPER
[ ] confluent-zookeeper: active (running) di cp-node1, cp-node2, cp-node3
[ ] echo srvr | nc <node> 2181 → menampilkan Mode dan Node count
[ ] 1 node Mode: leader, 2 node Mode: follower
[ ] Port 2181 LISTEN di semua node, port 2888 di leader, 3888 di semua node
[ ] quorumListenOnAllIPs=true di zookeeper.properties (fix deployment ini)

KAFKA BROKER
[ ] confluent-server: active (running) di cp-node1, cp-node2, cp-node3
[ ] Broker IDs terdaftar: [1, 2, 3] via zookeeper-shell
[ ] Port 9092 LISTEN di semua node
[ ] Kafka CLI dapat dijalankan dari salah satu data node

SCHEMA REGISTRY
[ ] confluent-schema-registry: active (running) di semua node
[ ] Port 8081 LISTEN dan merespons
[ ] curl /config → compatibilityLevel: BACKWARD
[ ] Schema dapat didaftarkan via REST API
[ ] Schema evolution (BACKWARD compatibility) berfungsi
[ ] Avro schema terdaftar otomatis saat produce via REST Proxy

KAFKA CONNECT
[ ] confluent-kafka-connect: active (running) di semua node
[ ] Port 8083 LISTEN, curl / → version 7.9.5-ce
[ ] Internal topics (connect-cluster-configs/offsets/status) tersedia
[ ] 4 connector plugins tersedia (Replicator, MirrorMaker2 x3)

KSQLDB
[ ] confluent-ksqldb: active (running) di semua node
[ ] Port 8088 LISTEN, curl /info → serverStatus: RUNNING
[ ] ksqlDB CLI dapat diakses dari data node
[ ] REST API query (SHOW TOPICS) berfungsi

REST PROXY
[ ] confluent-kafka-rest: active (running) di semua node
[ ] Port 8082 LISTEN
[ ] curl /topics → daftar topic tersedia
[ ] Produce JSON berhasil
[ ] Produce Avro berhasil (value_schema_id terdaftar di Schema Registry)
[ ] Consume via REST API berhasil

CONTROL CENTER
[ ] confluent-control-center: active (running) di cp-ansible
[ ] Port 9021 LISTEN, curl → HTTP 200
[ ] Web UI dapat diakses di http://192.168.56.10:9021
[ ] Cluster terdeteksi: 3 brokers, 59+ topics
[ ] ksqlDB clusters: 1, Connect clusters: 1

END-TO-END
[ ] Pipeline: REST Proxy → Kafka → ksqlDB filter → Consumer berfungsi
[ ] Fault tolerance: cluster berfungsi saat 1 node down
[ ] Recovery: semua service rejoin setelah node dinyalakan kembali
```
