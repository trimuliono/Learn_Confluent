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
  --topic avro-user \
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
Source:
https://docs.confluent.io/platform/7.9/schema-registry/index.html
https://docs.confluent.io/platform/7.9/schema-registry/serdes-develop/index.html
https://docs.confluent.io/platform/7.9/schema-registry/serdes-develop/serdes-avro.html#kafka-avro-console-producer
https://docs.confluent.io/platform/7.9/schema-registry/schema-compatibility.html
