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




---
Source:
https://docs.confluent.io/platform/7.9/schema-registry/index.html
https://docs.confluent.io/platform/7.9/schema-registry/serdes-develop/index.html
https://docs.confluent.io/platform/7.9/schema-registry/serdes-develop/serdes-avro.html#kafka-avro-console-producer
https://docs.confluent.io/platform/7.9/schema-registry/schema-compatibility.html
