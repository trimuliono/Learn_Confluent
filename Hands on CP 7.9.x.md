# Hands on CP 7.9.x

**dokumentasi ini akan mencakup 3 section hands on lab sebagai berikut:**
- Create kafka client to produce and consume data through a topic with avro schema
- Create connector using kafka connect (source and sink connector)
- Create stream processing using ksqlDB

---
## Create kafka client to produce and consume data through a topic with avro schema

Ini adalah **inti ekosistem Confluent**. Hampir semua fitur Confluent (Schema Registry, Connect, ksqlDB, governance) **berangkat dari sini**.

1️⃣ **Penjelasan Konsep (WHY sebelum HOW)**
**Apa yang akan kamu lakukan?**
Kamu akan:
1. Membuat topic Kafka
2. Menggunakan Schema Registry
3. Produce data ke Kafka dengan Avro format
4. Consume data dari Kafka dengan schema-aware consumer

**Kenapa Avro + Schema Registry?**
Dibanding JSON / String:
- Ada **schema enforcement**
- Ada **schema evolution** (backward / forward compatibility)
- Aman untuk multi producer & consumer
- Wajib untuk:
    - Kafka Connect
    - ksqlDB
    - Data governance (compatibility, versioning)

📌 **Di Confluent, Avro + Schema Registry itu “default enterprise pattern”**

2️⃣ **Arsitektur yang sedang kamu bangun**
```text
Kafka Producer
   |
   | (Avro + Schema ID)
   v
Kafka Topic  <---->  Schema Registry
   |
   v
Kafka Consumer
```
Yang dikirim ke Kafka **BUKAN schema**, tapi:
- **payload Avro (binary)**
- **schema ID** (lookup ke Schema Registry)


3️⃣ **Prerequisite (WAJIB sebelum lanjut)**

✅ A. Service harus RUNNING
Pastikan semua ini UP
```bash
systemctl status confluent-server
systemctl status confluent-schema-registry
systemctl status confluent-control-center
```
4️⃣ **Buat Topic Kafka**
```bash
kafka-topics --create --topic \
avro-user-demo --partitions 1 \
--replication-factor 1 \
--bootstrap-server localhost:9092
```
verifikasi:
```bash
kafka-topics --bootstrap-server localhost:9092 --list |grep avro
```
<img width="876" height="126" alt="image" src="https://github.com/user-attachments/assets/d94f080b-a151-427e-a950-500f3fee690e" />

5️⃣ **Jalankan Kafka Avro Console Producer**

Command utama:
```bash
kafka-avro-console-producer \
  --bootstrap-server localhost:9092 \
  --topic avro-user-demo \
  --property schema.registry.url=http://localhost:8085 \
  --property value.schema='
{
  "type": "record",
  "name": "User",
  "namespace": "com.example.avro",
  "fields": [
    {"name": "id", "type": "int"},
    {"name": "name", "type": "string"},
    {"name": "email", "type": ["null","string"], "default": null}
  ]
}'
```
📌 **Yang terjadi di belakang layar**
- Schema AUTO-REGISTER ke Schema Registry
- Topic belum berisi apa pun sampai kamu kirim data
   
6️⃣ **Produce data (ketik manual)**

Di prompt producer, kirim JSON sesuai schema:
```
{"id":1,"name":"tri","email":{"string":"tri@mail.com"}}
{"id":2,"name":"kafka-user","email":null}
{"id":3,"name":"confluent","email":{"string":"cp@confluent.io"}
```
> `Ctrl + D` → keluar producer

Di **Avro JSON encoding**, kalau field bertipe **union (["null","string"])**:
❌ TIDAK BOLEH langsung string
```
"email": "ihsan@mail.com"
```
✅ HARUS pakai wrapper union
```
"email": {"string": "ihsan@mail.com"}
```
dan untuk null
```
"email": null
```

7️⃣ **Cek Schema Registry (VALIDASI PENTING)**

**7.1 Lihat subject**
```
curl http://localhost:8085/subjects
```
Output:
```
["avro-user-demo-value"]
```
📌 **Naming rule**
```
<topic-name>-value
```

**7.2 Lihat schema version**
```
curl http://localhost:8085/subjects/avro-user-demo-value/versions
```
Output:
```
[1]
```

**7.3 Lihat schema detail**
```
curl --silent http://localhost:8085/subjects/avro-user-demo-value/versions/1 | jq
```

8️⃣ **Jalankan Kafka Avro Console Consumer**

```
kafka-avro-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic avro-user-demo \
  --from-beginning \
  --property schema.registry.url=http://localhost:8085
```
Output:
```
{"id":1,"name":"tri","email":{"string":"tri@mail.com"}}
{"id":2,"name":"kafka-user","email":null}
{"id":3,"name":"confluent","email":{"string":"cp@confluent.io"}}
```
🎯 **Avro berhasil end-to-end**

9️⃣ **Verifikasi dari Control Center (C3)**

1. Buka **Control Center**
2. Masuk cluster
3. Buka **Topics → avro-user-demo**
4. Lihat:
    - Messages count bertambah
    - Value format: **AVRO**
      
<img width="951" height="394" alt="image" src="https://github.com/user-attachments/assets/1e7a22af-eb0e-4bc7-a3b9-f676e93dc105" />

<img width="1888" height="937" alt="image" src="https://github.com/user-attachments/assets/5a2fc50b-9d6c-4116-ba8f-8c71ada966f0" />

<img width="942" height="465" alt="image" src="https://github.com/user-attachments/assets/7bcf84e0-1a16-45c2-878e-39cbd5c9a3dd" />

---
### Schema Evolution Avro (V2) – Confluent Platform 7.9
1️⃣ Tujuan Lab
Melakukan schema evolution pada Avro schema di Kafka menggunakan Schema Registry, tanpa merusak consumer lama.

Pada lab ini kita akan:

- Menambahkan field baru ke schema (age)
- Mengatur compatibility mode
- Membuktikan producer baru (v2) masih bisa dibaca consumer lama (v1)

2️⃣ Konsep Dasar (WAJIB PAHAM)
🔹 Apa itu Schema Evolution?

Schema Evolution adalah kemampuan untuk mengubah schema data (tambah/hapus/ubah field) tanpa memutus sistem yang sudah berjalan.

Kafka + Avro + Schema Registry menyediakan:

- Versioning schema
- Compatibility check
- Centralized schema management

🔹 Compatibility Mode (ringkas tapi penting)
| Mode     | Penjelasan                        |
| -------- | --------------------------------- |
| BACKWARD | Consumer lama bisa baca data baru |
| FORWARD  | Consumer baru bisa baca data lama |
| FULL     | Dua arah                          |
| NONE     | Tidak ada proteksi (⚠️ bahaya)    |

📌 Best practice default: BACKWARD

3️⃣ Kondisi Awal (Schema V1)
Schema awal yang sudah kamu gunakan:
```
{
  "type": "record",
  "name": "User",
  "namespace": "com.example.avro",
  "fields": [
    {"name": "id", "type": "int"},
    {"name": "name", "type": "string"},
    {"name": "email", "type": ["null","string"], "default": null}
  ]
}
```
👉 Ini akan tersimpan di Schema Registry sebagai:
```
Subject: avro-user-demo-value
Version: 1
```

4️⃣ Set Compatibility Mode (BACKWARD)
🔹 Cek compatibility saat ini
```
curl http://localhost:8085/config/avro-user-demo-value
```
Jika belum ada:
```json
{"compatibilityLevel":"BACKWARD"}
```
Jika mau set ulang (opsional tapi bagus untuk lab):
```
curl -X PUT \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"compatibility":"BACKWARD"}' \
  http://localhost:8085/config/avro-user-demo-value
```

5️⃣ Schema Evolution: Versi 2 (V2)
🔹 Perubahan yang dilakukan
Kita MENAMBAHKAN field baru:
```
{"name": "age", "type": ["null","int"], "default": null}
```
📌 Kenapa:
Type union dengan `null`
Ada `default`
➡️ Ini syarat BACKWARD compatibility

🔹 Schema V2 (lengkap)
```
{
  "type": "record",
  "name": "User",
  "namespace": "com.example.avro",
  "fields": [
    {"name": "id", "type": "int"},
    {"name": "name", "type": "string"},
    {"name": "email", "type": ["null","string"], "default": null},
    {"name": "age", "type": ["null","int"], "default": null}
  ]
}
```

6️⃣ Produce Data dengan Schema V2
```
kafka-avro-console-producer \
  --topic avro-user-demo \
  --bootstrap-server localhost:9092 \
  --property schema.registry.url=http://localhost:8085 \
  --property value.schema='
{
  "type": "record",
  "name": "User",
  "namespace": "com.example.avro",
  "fields": [
    {"name": "id", "type": "int"},
    {"name": "name", "type": "string"},
    {"name": "email", "type": ["null","string"], "default": null},
    {"name": "age", "type": ["null","int"], "default": null}
  ]
}'
```
Input data:
```
{"id":4,"name":"budi","email":{"string":"budi@mail.com"},"age":{"int":30}}
```
✔️ Schema Registry akan:
- Mendeteksi perubahan
- Register sebagai version 2

7️⃣ Verifikasi Schema Registry
🔹 List subject
```
curl http://localhost:8085/subjects
```
🔹 Lihat versi schema
```
curl http://localhost:8085/subjects/avro-user-demo-value/versions
```
🔹 Lihat versi schema
```
[1,2]
```
<img width="1880" height="937" alt="image" src="https://github.com/user-attachments/assets/3b071e52-be4d-4826-bdbe-ec405b0e6a73" />


8️⃣ Consume Data (Backward Compatibility Test)
```
kafka-avro-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic avro-user-demo \
  --from-beginning \
  --property schema.registry.url=http://localhost:8085
```
Output:
```
{"id":1,"name":"tri","email":{"string":"tri@mail.com"}}
{"id":2,"name":"kafka-user","email":null}
{"id":3,"name":"confluent","email":{"string":"cp@confluent.io"}}
{"id":4,"name":"budi","email":{"string":"budi@mail.com"},"age":{"int":30}}
```
> 📌  Consumer lama (schema lama) dapat membaca data yang diproduce dengan schema baru
<img width="1904" height="890" alt="image" src="https://github.com/user-attachments/assets/4a1a51c4-4296-4619-856c-05d3f3ce5416" />

<img width="1368" height="300" alt="image" src="https://github.com/user-attachments/assets/a1cdee47-fec3-4674-b844-08fe9d8017f9" />

---
Siap.
Di bawah ini **SATU DOKUMEN MARKDOWN UTUH**, **lengkap**, **rapi**, **siap copy–paste**, **tanpa ada teks di luar markdown**.
Langsung bisa kamu simpan sebagai file `.md`.

---


### 📘 LAB: Kafka Connect Source (Datagen) & Sink (JDBC PostgreSQL)
**Confluent Platform 7.9**

---

#### 1️⃣ Tujuan Lab

Pada lab ini peserta akan:

- Memahami arsitektur Kafka Connect
- Menjalankan Kafka Connect dalam Distributed Mode
- Membuat Source Connector menggunakan Datagen
- Membuat Sink Connector menggunakan JDBC ke PostgreSQL (remote server)
- Menggunakan Avro dan Schema Registry
- Memverifikasi alur data end-to-end
- Mengelola lifecycle connector (create, pause, resume, delete)

---

##### 2️⃣ Konsep Dasar Kafka Connect

##### 🔹 Apa itu Kafka Connect

Kafka Connect adalah framework bawaan Apache Kafka yang digunakan untuk:

- Mengambil data dari sistem eksternal ke Kafka (Source Connector)
- Mengirim data dari Kafka ke sistem eksternal (Sink Connector)
- Tanpa perlu menulis kode producer atau consumer secara manual

---

##### 🔹 Tipe Connector

| Tipe | Fungsi |
|-----|------|
| Source Connector | External system → Kafka |
| Sink Connector | Kafka → External system |

---

##### 🔹 Mode Kafka Connect

| Mode | Kegunaan |
|----|--------|
| Standalone | Development / Lab |
| Distributed | Production / High Availability |

> **Lab ini menggunakan Distributed Mode**

---

#### 3️⃣ Arsitektur Lab

```

Datagen Source Connector
↓ (Avro + Schema Registry)
Kafka Topic: kafka-connect-demo
↓
JDBC Sink Connector
↓
PostgreSQL (Remote Server)
Table: kafka_connect_demo

````

---

#### 4️⃣ Prerequisite

##### 🔹 Pastikan Service RUNNING

```bash
systemctl status confluent-server
systemctl status confluent-schema-registry
systemctl status confluent-control-center
systemctl status confluent-kafka-connect
````

---

##### 🔹 Cek Kafka Connect REST API

```bash
curl http://localhost:8083/connectors
```

Expected output:

```json
[]
```

---

##### 🔹 Cek Schema Registry

> Pada lab ini Schema Registry berjalan di **port 8085**

```bash
curl http://localhost:8085/subjects
```

---

#### 5️⃣ Buat Kafka Topic

```bash
kafka-topics \
  --bootstrap-server localhost:9092 \
  --create \
  --topic kafka-connect-demo \
  --partitions 1 \
  --replication-factor 1
```

---

#### 6️⃣ Install Kafka Connect Plugins

##### 🔹 Install Datagen Source Connector

```bash
sudo confluent-hub install confluentinc/kafka-connect-datagen:latest
```

Pilih:

* `1` → installed rpm/deb package
* `y` → update detected configs

---

##### 🔹 Install JDBC Sink Connector

```bash
sudo confluent-hub install confluentinc/kafka-connect-jdbc:latest
```

Pilih:

* `1` → installed rpm/deb package
* `y` → update detected configs

---

##### 🔹 Restart Kafka Connect

```bash
sudo systemctl restart confluent-kafka-connect
```

---

##### 🔹 Verifikasi Plugin Terinstall

```bash
curl --silent http://localhost:8083/connector-plugins | jq
```

Pastikan muncul:

* `io.confluent.kafka.connect.datagen.DatagenConnector`
* `io.confluent.connect.jdbc.JdbcSinkConnector`

---

#### 7️⃣ Source Connector – Datagen (Avro)

##### 🔹 Tujuan

Menghasilkan data dummy (users) dan mengirimkannya ke Kafka Topic menggunakan Avro dan Schema Registry.

---

##### 🔹 Config Datagen Source Connector

📄 **datagen-source-connector.json**

```json
{
  "name": "datagen-source-connect-demo",
  "config": {
    "connector.class": "io.confluent.kafka.connect.datagen.DatagenConnector",
    "tasks.max": "1",
    "kafka.topic": "kafka-connect-demo",
    "quickstart": "users",
    "iterations": "10",

    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://localhost:8085"
  }
}
```

---

##### 🔹 Create Datagen Source Connector

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  --data @datagen-source-connector.json \
  http://localhost:8083/connectors
```

---

##### 🔹 Cek Status Source Connector

```bash
curl http://localhost:8083/connectors/datagen-source-connect-demo/status | jq
```

> ⚠️ Jika task berstatus `FAILED` dengan pesan
> `generated the configured X number of messages`
> **Ini NORMAL**, Datagen berhenti setelah `iterations` terpenuhi.

---

#### 8️⃣ Verifikasi Data di Kafka

```bash
kafka-avro-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic kafka-connect-demo \
  --from-beginning \
  --property schema.registry.url=http://localhost:8085
```

Contoh output:

```json
{"registertime":151876119232,"userid":"User_9","regionid":"Region_8","gender":"FEMALE"}
```

---

#### 9️⃣ Sink Connector – JDBC PostgreSQL (Remote Server)

##### 🔹 Prerequisite Database

* PostgreSQL berada di server lain
* User database memiliki privilege:

  * CONNECT
  * CREATE
  * INSERT
  * USAGE pada schema (misalnya `public`)

> **Tabel TIDAK perlu dibuat manual**
> JDBC Sink akan membuat tabel otomatis (`auto.create=true`)

---

##### 🔹 Config JDBC Sink Connector

📄 **jdbc-sink-connector.json**

```json
{
  "name": "jdbc-sink-postgres-demo",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "tasks.max": "1",
    "topics": "kafka-connect-demo",

    "connection.url": "jdbc:postgresql://10.100.13.205:5432/ihsan",
    "connection.user": "ihsan",
    "connection.password": "ihsan",

    "auto.create": "true",
    "auto.evolve": "true",

    "insert.mode": "insert",
    "pk.mode": "none",

    "table.name.format": "kafka_connect_demo",

    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://localhost:8085"
  }
}
```

---

##### 🔹 Create JDBC Sink Connector

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  --data @jdbc-sink-connector.json \
  http://localhost:8083/connectors
```

---

##### 🔹 Cek Status Sink Connector

```bash
curl http://localhost:8083/connectors/jdbc-sink-postgres-demo/status | jq
```

Expected:

```json
"state": "RUNNING"
```

---

#### 🔟 Verifikasi Data di PostgreSQL

```bash
psql -h 10.100.13.205 -U ihsan -d ihsan
```

```sql
\d kafka_connect_demo;
SELECT * FROM kafka_connect_demo;
```

---

#### 1️⃣1️⃣ Mengelola Lifecycle Connector

##### 🔹 Pause Connector

```bash
curl -X PUT http://localhost:8083/connectors/datagen-source-connect-demo/pause
```

---

##### 🔹 Resume Connector

```bash
curl -X PUT http://localhost:8083/connectors/datagen-source-connect-demo/resume
```

---

##### 🔹 Delete Connector

```bash
curl -X DELETE http://localhost:8083/connectors/datagen-source-connect-demo
```

---

## 1️⃣2️⃣ Troubleshooting

##### ❌ Sink tidak membuat tabel

* User database tidak punya privilege CREATE
* Schema bukan `public`
* Salah `connection.url`

---

##### ❌ Error Avro / Schema not found

* Schema Registry tidak RUNNING
* Port Schema Registry salah
* Subject terhapus

---

##### ❌ Datagen task FAILED

* Normal jika `iterations` sudah habis
* Connector tetap bisa dihapus atau di-pause

---
## 📘 LAB: Stream Processing using ksqlDB Confluent Platform 7.9

### 1️⃣ Tujuan Lab

Pada lab ini peserta akan:

- Memahami konsep stream processing di Kafka
- Menggunakan ksqlDB untuk query data Kafka secara real-time
- Membuat STREAM dari topic Kafka (Avro)
- Melakukan filtering, projection, dan aggregation
- Membuat STREAM dan TABLE hasil transformasi
- Memverifikasi data secara real-time

### 2️⃣ Konsep Dasar ksqlDB

#### 🔹 Apa itu ksqlDB

ksqlDB adalah engine SQL streaming untuk Kafka yang memungkinkan kita:

- Query data Kafka menggunakan SQL
- Membuat stream & table tanpa menulis Java code
- Melakukan transformasi data real-time
- Menyimpan hasil transformasi kembali ke Kafka

#### 🔹 Perbedaan STREAM vs TABLE

| Konsep | Penjelasan |
|-----|-----------|
| STREAM | Event flow (append-only), cocok untuk event log |
| TABLE | State (latest value per key), hasil agregasi |

### 3️⃣ Arsitektur Lab
```
Datagen Source
↓
Kafka Topic (Avro)
kafka-connect-demo
↓
ksqlDB
├─ STREAM users_stream
├─ STREAM filtered_stream
└─ TABLE gender_count
```
### 4️⃣ Prerequisite

Pastikan service berikut **RUNNING**:

```bash
systemctl status confluent-server
systemctl status confluent-schema-registry
systemctl status confluent-kafka-connect
systemctl status confluent-ksqldb
```

####🔹 Verifikasi ksqlDB Server
```
ccurl --silent http://localhost:8088/info | jq
```
<img width="884" height="234" alt="image" src="https://github.com/user-attachments/assets/e3c477f9-6d77-4e59-b340-c6707ca98f34" />

### 5️⃣ Masuk ke ksqlDB CLI
```
ksql http://localhost:8088
```
jika berhasil akan muncul prompt
```
ksql>
```
<img width="1106" height="644" alt="image" src="https://github.com/user-attachments/assets/d3ae405b-1e4c-4cf9-91f8-5153e961506a" />

### 6️⃣ Set ksqlDB Properties (WAJIB)
Agar ksqlDB bisa membaca Avro dari Schema Registry:
```
SET 'auto.offset.reset' = 'earliest';
SET 'ksql.schema.registry.url' = 'http://localhost:8085';
```
<img width="1655" height="143" alt="image" src="https://github.com/user-attachments/assets/7fc1dcf1-08a9-4b0e-9967-759e9c47a4b1" />

### 7️⃣ Verifikasi Topic Kafka
```sql
SHOW TOPICS;
```
Pastikan topic berikut muncul:
```
kafka-connect-demo
```
<img width="798" height="276" alt="image" src="https://github.com/user-attachments/assets/e8030ec5-bc46-484d-a835-392b863a80a5" />

### 8️⃣ Buat STREAM dari Topic Kafka (Avro)

#### 🔹 Buat STREAM `users_stream`
```
CREATE STREAM users_stream (
  registertime BIGINT,
  userid STRING,
  regionid STRING,
  gender STRING
)
WITH (
  KAFKA_TOPIC = 'kafka-connect-demo',
  VALUE_FORMAT = 'AVRO'
);
```
#### 🔹 Verifikasi STREAM
```sql
SHOW STREAMS;

DESCRIBE users_stream;
```
<img width="1149" height="627" alt="image" src="https://github.com/user-attachments/assets/9cb1aa46-b732-42f6-93c4-e7c89b3ac743" />

#### 🔹 Query Data Real-Time
```sql
SELECT * FROM users_stream EMIT CHANGES;
```
Tekan `Ctrl + C` untuk keluar dari query.
<img width="1814" height="524" alt="image" src="https://github.com/user-attachments/assets/cec9f8b2-2e09-41a1-9b64-79ae305f3e8c" />

### 9️⃣ Filtering Data (STREAM → STREAM)

#### 🔹 Buat STREAM hanya untuk gender FEMALE
```sql
CREATE STREAM female_users AS
SELECT *
FROM users_stream
WHERE gender = 'FEMALE'
EMIT CHANGES;
```
####🔹 Query hasil filter
```sql
SELECT * FROM female_users EMIT CHANGES;
```
<img width="1523" height="669" alt="image" src="https://github.com/user-attachments/assets/a36d2bef-807d-4c1f-9b87-b5300d63cfd8" />

### 🔟 Projection (Pilih Kolom Tertentu)
```sql
CREATE STREAM user_basic_info AS
SELECT userid, regionid
FROM users_stream
EMIT CHANGES;
```
<img width="530" height="175" alt="image" src="https://github.com/user-attachments/assets/e5dbb23b-cf5a-46ac-8485-9b9015d0f553" />

### 1️⃣1️⃣ Aggregation (STREAM → TABLE)

#### 🔹 Hitung jumlah user per gender
```
CREATE TABLE gender_count AS
SELECT gender,
       COUNT(*) AS total
FROM users_stream
GROUP BY gender
EMIT CHANGES;
```

#### 🔹 Query TABLE
```
SELECT * FROM gender_count EMIT CHANGES;
```
<img width="1868" height="603" alt="image" src="https://github.com/user-attachments/assets/819eda41-6d2c-47ab-a42d-2b19ee566ea7" />
> TABLE akan menampilkan **state terbaru**, bukan semua event

### 1️⃣2️⃣ Sink Hasil ksqlDB ke Kafka
ksqlDB otomatis membuat topic baru untuk hasil STREAM/TABLE.

Cek topic:
```
SHOW TOPICS;
```
<img width="675" height="300" alt="image" src="https://github.com/user-attachments/assets/10f6d7a5-3ad3-4d5e-b7c0-3d4ee8d3f033" />

### 1️⃣3️⃣ Cleanup (Opsional)

#### 🔹 Drop STREAM
```
DROP STREAM female_users DELETE TOPIC;
DROP STREAM user_basic_info DELETE TOPIC;
```

#### 🔹 Drop TABLE
```
DROP TABLE gender_count DELETE TOPIC;
```

### 1️⃣4️⃣ Exit ksqlDB CLI
```
EXIT;
# atau tekan Ctrl + D
```

### 1️⃣5️⃣ Troubleshooting

❌ Tidak bisa baca Avro
- Schema Registry tidak running
- URL Schema Registry salah
- VALUE_FORMAT bukan AVRO

❌ Data tidak muncul
- Offset belum earliest
- Topic kosong
- Producer sudah berhenti (iterations habis atau di pause)


---
Source:
https://docs.confluent.io/platform/7.9/schema-registry/index.html
https://docs.confluent.io/platform/7.9/schema-registry/serdes-develop/index.html
https://docs.confluent.io/platform/7.9/schema-registry/serdes-develop/serdes-avro.html#kafka-avro-console-producer
https://docs.confluent.io/platform/7.9/schema-registry/schema-compatibility.html
https://docs.confluent.io/platform/7.9/connect/index.html
https://docs.confluent.io/platform/7.9/ksqldb/index.html
https://docs.confluent.io/platform/7.9/ksqldb/concepts.html
https://docs.confluent.io/platform/7.9/ksqldb/developer-guide/ksqldb-reference.html
