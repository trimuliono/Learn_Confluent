# 🔐 LAB – Kafka Security End-to-End

## Confluent Platform 7.9

### (Security Extension from Phase 1-2 ZooKeeper Node Single Host)

---

# 📌 Scope Lab

Lab ini melanjutkan Phase 1 yang sudah memiliki:

- 3 ZooKeeper nodes (2181, 2182, 2183)
- 1 Kafka Broker
- Control Center
- Schema Registry
- Kafka Connect

Lab ini fokus menambahkan security **dari layer terdalam ke luar** (inside-out approach):

| Step | Fokus | Alasan Urutan |
|------|-------|---------------|
| 1 | ZooKeeper Quorum Security | Fondasi cluster, harus diamankan pertama |
| 2 | ZooKeeper Client + Kafka↔ZK Connection | Koneksi ke fondasi harus secure sebelum broker |
| 3 | Kafka Inter-Broker Security | Komunikasi internal broker |
| 4 | Kafka Client Security | Koneksi producer/consumer ke broker |
| 5 | ACL Authorization | Authorization di atas authentication |
| 6 | Full Testing Produce & Consume | Validasi end-to-end |
| 7 | Summary & Documentation | Dokumentasi seluruh proses |

> **Prinsip Utama:** Amankan dari dalam ke luar. Jika broker sudah secure tapi ZooKeeper belum, ada gap security di fondasi cluster. Jika ZooKeeper di-secure belakangan, broker bisa kehilangan koneksi ke ZK dan cluster down.

---

# 🧠 1️⃣ SECURITY CONCEPT (WAJIB PAHAM SEBELUM MULAI)

## Mengapa Kafka Perlu Security?

Kafka secara default berjalan **tanpa security sama sekali** — tidak ada enkripsi, tidak ada authentication, tidak ada authorization. Ini artinya:

- **Tanpa Encryption (SSL/TLS):** Semua data yang lewat antara producer, broker, consumer, dan ZooKeeper bisa di-sniff menggunakan tools seperti Wireshark atau tcpdump. Password, data bisnis, PII — semua terbaca plaintext di network.
- **Tanpa Authentication (SASL):** Siapa saja yang bisa reach network Kafka bisa langsung produce/consume data. Tidak ada mekanisme "siapa kamu?" — semua dianggap trusted.
- **Tanpa Authorization (ACL):** Bahkan jika sudah ada authentication, semua user punya akses penuh. User biasa bisa delete topic, baca data sensitif, atau menulis data sampah ke topic production.

## Layer Security Kafka

| Layer | Protokol | Fungsi | Analogi |
|-------|----------|--------|---------|
| Encryption | SSL/TLS | Encrypt semua traffic di wire | Amplop tertutup — isi surat tidak bisa dibaca orang lain |
| Authentication | SASL | Verifikasi identitas client/broker | KTP — membuktikan siapa kamu |
| Authorization | ACL | Kontrol akses per resource | Kartu akses gedung — KTP saja tidak cukup, perlu izin masuk ruangan tertentu |
| ZooKeeper Security | SSL + SASL | Amankan metadata cluster | Brankas — tempat penyimpanan konfigurasi cluster harus terkunci |

## Kombinasi Protokol Security

| Protocol | Encryption | Authentication |
|----------|-----------|---------------|
| PLAINTEXT | ❌ | ❌ |
| SSL | ✅ | ❌ |
| SASL_PLAINTEXT | ❌ | ✅ |
| SASL_SSL | ✅ | ✅ |

> **Best Practice Production:** Gunakan `SASL_SSL` untuk mendapatkan encryption + authentication sekaligus.

## Konsep SSL/TLS di Kafka

SSL/TLS di Kafka menggunakan **mutual TLS** atau **one-way TLS**:

- **Keystore:** Menyimpan private key dan certificate milik sendiri. Ini identitas si pemilik.
- **Truststore:** Menyimpan certificate CA yang dipercaya. Ini daftar "siapa yang saya percaya."
- **CA (Certificate Authority):** Pihak yang menandatangani certificate. Semua pihak harus percaya CA yang sama.

Alur handshake SSL:
```
Client                          Broker
  |--- ClientHello ------------->|
  |<-- ServerHello + Cert -------|
  |    (client verifikasi cert   |
  |     broker via truststore)   |
  |--- Client Cert (optional) -->|
  |    (broker verifikasi cert   |
  |     client via truststore)   |
  |<-- Encrypted Session ------->|
```

## Konsep SASL di Kafka

SASL (Simple Authentication and Security Layer) mendukung beberapa mekanisme:

| Mekanisme | Deskripsi | Use Case |
|-----------|-----------|----------|
| PLAIN | Username/password plaintext (harus pakai SSL) | Development, simple setup |
| SCRAM-SHA-256/512 | Username/password dengan hashing | Production tanpa Kerberos |
| GSSAPI (Kerberos) | Ticket-based authentication | Enterprise dengan Active Directory |
| OAUTHBEARER | Token-based authentication | Cloud-native, modern apps |

> **Lab ini menggunakan SASL/PLAIN** — paling sederhana untuk belajar konsep. Di production, gunakan SCRAM atau Kerberos.

## Konsep ACL di Kafka

ACL (Access Control List) mengontrol **siapa boleh melakukan apa pada resource mana**.

Struktur ACL:
```
Principal + Permission + Operation + Resource = Rule
```

Contoh:
```
User:user1 + ALLOW + WRITE + Topic:orders = user1 boleh menulis ke topic orders
User:user2 + DENY  + READ  + Topic:secret = user2 dilarang baca topic secret
```

Prinsip penting:
- **Deny overrides Allow** — jika ada DENY dan ALLOW untuk operasi yang sama, DENY menang.
- **No ACL = Deny** (jika `allow.everyone.if.no.acl.found=false`) — default deny.
- **Super User** — bypass semua ACL, digunakan untuk admin.

---

# 🔒 2️⃣ ZOOKEEPER QUORUM SECURITY (Encryption & Authentication)

## Mengapa ZooKeeper Harus Diamankan Pertama?

ZooKeeper adalah **otak dari Kafka cluster**. Ia menyimpan:

- Metadata broker (broker mana yang aktif, siapa controller)
- Konfigurasi topic (partisi, replication factor, retention)
- Consumer group offsets (pada versi lama)
- ACL data
- Controller election state

Jika ZooKeeper tidak secure:
- **Attacker bisa membaca semua metadata cluster** — tahu struktur topic, konfigurasi, dll.
- **Attacker bisa memodifikasi konfigurasi** — mengubah replication factor, delete topic, dll.
- **Quorum communication bisa di-intercept** — data sinkronisasi antar ZK node terbaca.

ZooKeeper quorum security mengamankan **komunikasi antar ZooKeeper nodes** (port 2888 untuk follower dan 3888 untuk election).

## 2.1 Buat Folder Secret

```bash
sudo mkdir -p /etc/kafka/secrets
cd /etc/kafka/secrets
```

## 2.2 Generate CA (Certificate Authority)

CA adalah root of trust — semua certificate akan ditandatangani oleh CA ini.

```bash
sudo openssl req -new -x509 \
  -keyout ca-key \
  -out ca-cert \
  -days 365 \
  -nodes \
  -subj "/CN=KafkaLabCA/O=Lab/L=Jakarta/ST=DKI/C=ID"
```
<img width="1527" height="554" alt="image" src="https://github.com/user-attachments/assets/0a670583-df7f-43f1-a172-a86789c8ee02" />

**Penjelasan parameter:**

| Parameter | Fungsi |
|-----------|--------|
| `-new -x509` | Buat self-signed certificate baru |
| `-keyout ca-key` | Simpan private key CA ke file `ca-key` |
| `-out ca-cert` | Simpan certificate CA ke file `ca-cert` |
| `-days 365` | Masa berlaku 1 tahun |
| `-nodes` | No DES — private key tidak di-encrypt (untuk lab) |
| `-subj` | Subject/identitas certificate |

**Verifikasi CA:**

```bash
sudo openssl x509 -in ca-cert -text -noout | head -20
```

Output yang diharapkan:
```
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: ...
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = KafkaLabCA, O = Lab, L = Jakarta, ST = DKI, C = ID
        Validity
            Not Before: ...
            Not After : ...
        Subject: CN = KafkaLabCA, O = Lab, L = Jakarta, ST = DKI, C = ID
```
<img width="1220" height="514" alt="image" src="https://github.com/user-attachments/assets/c59ce87b-0549-4587-a971-500ad3d3d956" />

## 2.3 Generate ZooKeeper Keystore & Truststore

Buat keystore dan truststore untuk **setiap ZooKeeper node**.

### Generate Keystore (per node)

```bash
# Node 1
sudo keytool -keystore zk1.keystore.jks \
  -alias zk1 \
  -validity 365 \
  -genkey -keyalg RSA \
  -dname "CN=localhost,O=Lab,L=Jakarta,ST=DKI,C=ID" \
  -storepass password \
  -keypass password

# Node 2
sudo keytool -keystore zk2.keystore.jks \
  -alias zk2 \
  -validity 365 \
  -genkey -keyalg RSA \
  -dname "CN=localhost,O=Lab,L=Jakarta,ST=DKI,C=ID" \
  -storepass password \
  -keypass password

# Node 3
sudo keytool -keystore zk3.keystore.jks \
  -alias zk3 \
  -validity 365 \
  -genkey -keyalg RSA \
  -dname "CN=localhost,O=Lab,L=Jakarta,ST=DKI,C=ID" \
  -storepass password \
  -keypass password
```
<img width="878" height="98" alt="image" src="https://github.com/user-attachments/assets/03869b6d-cd87-4323-898e-d1240d7b1bf0" />

### Create CSR, Sign, dan Import (per node)

Lakukan untuk setiap node. Contoh Node 1:

```bash
# Create CSR
sudo keytool -keystore zk1.keystore.jks \
  -alias zk1 \
  -certreq -file zk1.csr \
  -storepass password

# Sign dengan CA
sudo openssl x509 -req \
  -CA ca-cert -CAkey ca-key \
  -in zk1.csr \
  -out zk1-signed.crt \
  -days 365 -CAcreateserial

# Import CA cert ke keystore
sudo keytool -keystore zk1.keystore.jks \
  -alias CARoot -import -file ca-cert \
  -storepass password -noprompt

# Import signed cert ke keystore
sudo keytool -keystore zk1.keystore.jks \
  -alias zk1 -import -file zk1-signed.crt \
  -storepass password -noprompt
```
<img width="1207" height="569" alt="image" src="https://github.com/user-attachments/assets/11afe192-79f9-4cfb-a011-f74faff42aef" />

Ulangi langkah yang sama untuk Node 2 (`zk2`) dan Node 3 (`zk3`).

### Generate Truststore (shared untuk semua node)

```bash
sudo keytool -keystore zk.truststore.jks \
  -alias CARoot -import -file ca-cert \
  -storepass password -noprompt
```
<img width="707" height="89" alt="image" src="https://github.com/user-attachments/assets/b7faf4c7-3529-4d62-a629-945b30998656" />

**Verifikasi keystore dan truststore:**

```bash
# List isi keystore
keytool -list -keystore zk1.keystore.jks -storepass password

# List isi truststore
keytool -list -keystore zk.truststore.jks -storepass password
```
<img width="1611" height="440" alt="image" src="https://github.com/user-attachments/assets/f758f9d1-8e07-4a1a-8b2d-78dde8a72b86" />

## 2.4 Buat JAAS File untuk ZooKeeper

File JAAS mendefinisikan credential untuk authentication antar ZooKeeper nodes.

### File `/etc/kafka/zk_server_jaas.conf` (untuk ZK server process):

```properties
Server {
  org.apache.zookeeper.server.auth.DigestLoginModule required
  user_zkadmin="zkadmin-secret";
};

QuorumServer {
  org.apache.zookeeper.server.auth.DigestLoginModule required
  user_zkadmin="zkadmin-secret";
};

QuorumLearner {
  org.apache.zookeeper.server.auth.DigestLoginModule required
  username="zkadmin"
  password="zkadmin-secret";
};
```
<img width="898" height="301" alt="image" src="https://github.com/user-attachments/assets/14b4817b-a4d0-4874-891a-516f8761bdce" />

**Penjelasan setiap section:**

| Section | Fungsi |
|---------|--------|
| `Server` | Authentication untuk client yang connect ke ZooKeeper |
| `QuorumServer` | Authentication ketika node lain connect ke node ini |
| `QuorumLearner` | Credential yang digunakan node ini untuk connect ke node lain |

### File `/etc/kafka/zk_client_jaas.conf` (untuk testing client):

```properties
Client {
  org.apache.zookeeper.server.auth.DigestLoginModule required
  username="zkadmin"
  password="zkadmin-secret";
};
```

> **Penting:** `zk_server_jaas.conf` untuk ZK server process. `zk_client_jaas.conf` untuk client yang connect ke ZK (termasuk `zookeeper-shell`). Keduanya file terpisah karena section JAAS yang dibutuhkan berbeda.

## 2.5 Update zookeeper.properties (Setiap Node)

> **Catatan penting tentang konfigurasi tambahan:**
> - `secureClientPort` menambahkan port SSL **terpisah** dari `clientPort` (plain). Keduanya aktif bersamaan — plain port tetap ada tapi dilindungi SASL enforcement.
> - `serverCnxnFactory=...NettyServerCnxnFactory` **wajib** karena NIO (default) tidak support SSL. Tanpa ini, ZK crash dengan `SSL isn't supported in NIOServerCnxn`.
> - `ssl.keyStore.*` dan `ssl.trustStore.*` (tanpa prefix `quorum.`) diperlukan untuk **client SSL port**. Ini berbeda dari `ssl.quorum.keyStore.*` yang hanya untuk komunikasi antar ZK nodes.
> - `ssl.clientAuth=none` artinya server tidak meminta client certificate (one-way TLS).

Tambahkan konfigurasi berikut ke setiap `zookeeper.properties`:

### Node 1 (`/etc/kafka/zookeeper1.properties`)

```properties
# === SSL Quorum (antar ZooKeeper nodes) ===
sslQuorum=true
ssl.quorum.keyStore.location=/etc/kafka/secrets/zk1.keystore.jks
ssl.quorum.keyStore.password=password
ssl.quorum.trustStore.location=/etc/kafka/secrets/zk.truststore.jks
ssl.quorum.trustStore.password=password

# === SASL Authentication ===
authProvider.1=org.apache.zookeeper.server.auth.SASLAuthenticationProvider
requireClientAuthScheme=sasl
enforce.auth.enabled=true
enforce.auth.schemes=sasl

# === Secure Client Port (SSL) ===
secureClientPort=2281
serverCnxnFactory=org.apache.zookeeper.server.NettyServerCnxnFactory
ssl.clientAuth=none
ssl.keyStore.location=/etc/kafka/secrets/zk1.keystore.jks
ssl.keyStore.password=password
ssl.trustStore.location=/etc/kafka/secrets/zk.truststore.jks
ssl.trustStore.password=password
```
<img width="935" height="255" alt="image" src="https://github.com/user-attachments/assets/98338315-66c0-487f-b8a8-2a12f65c1f88" />

<img width="801" height="351" alt="image" src="https://github.com/user-attachments/assets/f7914b2e-b45c-46b5-a04a-3cf267c3f948" />

### Node 2 (`/etc/kafka/zookeeper2.properties`)

```properties
# === SSL Quorum (antar ZooKeeper nodes) ===
sslQuorum=true
ssl.quorum.keyStore.location=/etc/kafka/secrets/zk2.keystore.jks
ssl.quorum.keyStore.password=password
ssl.quorum.trustStore.location=/etc/kafka/secrets/zk.truststore.jks
ssl.quorum.trustStore.password=password

# === SASL Authentication ===
authProvider.1=org.apache.zookeeper.server.auth.SASLAuthenticationProvider
requireClientAuthScheme=sasl
enforce.auth.enabled=true
enforce.auth.schemes=sasl

# === Secure Client Port (SSL) ===
secureClientPort=2282
serverCnxnFactory=org.apache.zookeeper.server.NettyServerCnxnFactory
ssl.clientAuth=none
ssl.keyStore.location=/etc/kafka/secrets/zk2.keystore.jks
ssl.keyStore.password=password
ssl.trustStore.location=/etc/kafka/secrets/zk.truststore.jks
ssl.trustStore.password=password
```
<img width="810" height="705" alt="image" src="https://github.com/user-attachments/assets/82baa24f-be68-424b-9b8b-eeb11041902f" />

### Node 3 (`/etc/kafka/zookeeper3.properties`)

```properties
# === SSL Quorum (antar ZooKeeper nodes) ===
sslQuorum=true
ssl.quorum.keyStore.location=/etc/kafka/secrets/zk3.keystore.jks
ssl.quorum.keyStore.password=password
ssl.quorum.trustStore.location=/etc/kafka/secrets/zk.truststore.jks
ssl.quorum.trustStore.password=password

# === SASL Authentication ===
authProvider.1=org.apache.zookeeper.server.auth.SASLAuthenticationProvider
requireClientAuthScheme=sasl
enforce.auth.enabled=true
enforce.auth.schemes=sasl

# === Secure Client Port (SSL) ===
secureClientPort=2283
serverCnxnFactory=org.apache.zookeeper.server.NettyServerCnxnFactory
ssl.clientAuth=none
ssl.keyStore.location=/etc/kafka/secrets/zk3.keystore.jks
ssl.keyStore.password=password
ssl.trustStore.location=/etc/kafka/secrets/zk.truststore.jks
ssl.trustStore.password=password
```
<img width="827" height="708" alt="image" src="https://github.com/user-attachments/assets/13a158d6-71cf-4439-9bdf-c06c0cea2acd" />

### Perbedaan Property SSL di ZooKeeper

| Property | Scope | Untuk |
|----------|-------|-------|
| `ssl.quorum.keyStore.*` | Quorum | Komunikasi antar ZK nodes (port 2888/3888) |
| `ssl.quorum.trustStore.*` | Quorum | Verifikasi certificate ZK nodes lain |
| `ssl.keyStore.*` | Client | Secure client port (secureClientPort) |
| `ssl.trustStore.*` | Client | Verifikasi certificate client (jika clientAuth=need) |

## 2.6 Set Environment Variable untuk ZooKeeper

Edit file `/etc/default/zookeeper` (atau service file masing-masing node):

```bash
KAFKA_OPTS="-Djava.security.auth.login.config=/etc/kafka/zk_server_jaas.conf"
```
> **Karna kita menggunakan 3 node (3 port berbeda) dalam 1 server, kita menjalankan zookeeper dengan manual command**
> **Sehingga untuk `Set Environment Variable` kita cukup export langsung tepat sebelum melakukan restart service zookeeper**

## 2.7 Restart Semua ZooKeeper Nodes

```bash
export KAFKA_OPTS="-Djava.security.auth.login.config=/etc/kafka/zk_server_jaas.conf"

# Restart satu per satu, mulai dari follower
# Start masing-masing di background dengan -daemon
sudo -E /usr/bin/zookeeper-server-start -daemon /etc/kafka/zookeeper1.properties
sudo -E /usr/bin/zookeeper-server-start -daemon /etc/kafka/zookeeper2.properties
sudo -E /usr/bin/zookeeper-server-start -daemon /etc/kafka/zookeeper3.properties
```

> **Penting:** Restart secara rolling — satu node selesai dulu, baru restart node berikutnya. Jangan restart semua sekaligus karena quorum bisa hilang.
> `sudo -E` → flag `-E` meneruskan environment variable (`KAFKA_OPTS`) ke proses sudo. Tanpa `-E`, sudo akan strip environment variable dan JAAS tidak ter-load.
> `-daemon` → menjalankan ZooKeeper di background. Tanpa ini, terminal akan terkunci di node pertama dan tidak bisa start node 2 dan 3 di terminal yang sama dan harus membuat sesi terminal baru.

**Verifikasi JAAS Ter-load**
```
# Cek apakah KAFKA_OPTS masuk ke proses Java
ps aux | grep zookeeper | grep jaas
sudo ss -tlnp | grep -E "2181|2182|2183|2281|2282|2283"
```
**Expected:** Muncul 3 proses Java dengan `-Djava.security.auth.login.config=/etc/kafka/zk_server_jaas.conf`.
Kalau kosong, berarti `-E` tidak bekerja. Alternatifnya, edit langsung di start script `/usr/bin/zookeeper-server-start` tepat baris terakhir sebelum baris command `exec`.

<img width="778" height="396" alt="image" src="https://github.com/user-attachments/assets/def976ef-5d18-48ed-9baf-09e520fcb899" />
<img width="1045" height="232" alt="image" src="https://github.com/user-attachments/assets/78ec1cff-a704-4f78-adc4-ddf8c8942e27" />


---

## 🧪 2.8 Testing ZooKeeper Quorum Security

### Test 1 — Verifikasi Quorum & Leader Election ✅

```bash
for port in 2181 2182 2183; do echo "Port $port: $(echo ruok | nc localhost $port)"; done
```

**Expected:** Setiap node merespons `imok`.

Verifikasi via log:
```bash
sudo grep -i "ssl handshake complete\|LEADING\|FOLLOWING\|Using TLS encrypted" /var/log/kafka/zookeeper.out | tail -10
```

**Actual result:**
```
INFO Using TLS encrypted quorum communication
INFO SSL handshake complete ... TLSv1.2 - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
INFO Accepted TLS connection from localhost/127.0.0.1
INFO FOLLOWING - LEADER ELECTION TOOK - 647 MS
```
<img width="1865" height="283" alt="image" src="https://github.com/user-attachments/assets/1164359c-837c-45e8-9ca5-6eb0f7cc931f" />


### Test 2 — Plain Connection Tanpa SASL (Harus Ditolak) ✅

```bash
unset KAFKA_OPTS
zookeeper-shell localhost:2181
```

**Actual result:**
```
WatchedEvent state:AuthFailed type:None path:null
Session ... Closing socket connection
```
<img width="1860" height="374" alt="image" src="https://github.com/user-attachments/assets/0da364ed-f9f9-445a-bc6d-85ed02b9d32a" />

Server log menunjukkan:
```
ERROR Client authentication scheme(s) [ip] does not match with any of the expected authentication scheme [sasl], closing session.
```
<img width="1857" height="434" alt="image" src="https://github.com/user-attachments/assets/df7af05b-cd43-4e14-a60f-af2c50020a6e" />

> Client tanpa SASL credential hanya membawa scheme `[ip]`. ZK membutuhkan `[sasl]` → session ditolak. **SASL enforcement bekerja.**

### Test 3 — Plain Connection Dengan SASL (Harus Berhasil) ✅

```bash
KAFKA_OPTS="-Djava.security.auth.login.config=/etc/kafka/zk_client_jaas.conf" zookeeper-shell localhost:2181
```

**Actual result:**
```
WatchedEvent state:SyncConnected type:None path:null
WatchedEvent state:SaslAuthenticated type:None path:null
ls /
[admin, brokers, cells, cluster, cluster_links, config, consumers, controller_epoch,
 feature, isr_change_notification, latest_producer_id_block, leadership_priority,
 log_dir_event_notification, tenants, zookeeper]
```
<img width="1852" height="332" alt="image" src="https://github.com/user-attachments/assets/9ebcc348-779f-4125-80d1-920345361fcc" />

> **`SaslAuthenticated`** membuktikan SASL berhasil. `ls /` menampilkan semua znodes — client dengan credential yang benar dapat mengakses data ZooKeeper.

### Test 4 — Verifikasi SSL pada Secure Client Port ✅

```bash
echo "" | sudo openssl s_client -connect localhost:2281 -CAfile /etc/kafka/secrets/ca-cert 2>&1 | grep -i "verify\|subject\|issuer\|handshake"
```

**Actual result:**
```
verify return:1
subject=C = ID, ST = DKI, L = Jakarta, O = Lab, CN = localhost
issuer=CN = KafkaLabCA, O = Lab, L = Jakarta, ST = DKI, C = ID
SSL handshake has read 4541 bytes and written 386 bytes
    Verify return code: 0 (ok)
```
<img width="1752" height="164" alt="image" src="https://github.com/user-attachments/assets/c4b502f8-95fc-4431-b5bb-dd59f7e378a5" />

> Port 2281 serve SSL dengan certificate chain valid. `Verify return code: 0 (ok)`.

### Test 5 — ZooKeeper Shell via Secure Port Dengan SASL (Harus Berhasil) ✅

```bash
KAFKA_OPTS="
-Djava.security.auth.login.config=/etc/kafka/zk_client_jaas.conf
-Dzookeeper.client.secure=true
-Dzookeeper.clientCnxnSocket=org.apache.zookeeper.ClientCnxnSocketNetty
-Dzookeeper.ssl.protocol=TLSv1.2
-Dzookeeper.ssl.trustStore.location=/etc/kafka/secrets/zk.truststore.jks
-Dzookeeper.ssl.trustStore.password=password
-Dzookeeper.ssl.keyStore.location=/etc/kafka/secrets/zk1.keystore.jks
-Dzookeeper.ssl.keyStore.password=password
" zookeeper-shell localhost:2281
```
<img width="1852" height="462" alt="image" src="https://github.com/user-attachments/assets/1b035937-f6bc-4ec1-93ce-ad508fbdb353" />

**Expected:** Berhasil connect dan bisa jalankan `ls /`.
---
## 🔧 2.9 Troubleshooting yang Ditemukan Selama Lab

### Issue 1: `SSL isn't supported in NIOServerCnxn`

**Gejala:** ZK crash saat start setelah menambahkan `secureClientPort`.

**Solusi:** Tambahkan `serverCnxnFactory=org.apache.zookeeper.server.NettyServerCnxnFactory`.

### Issue 2: `No JAAS configuration section named 'Client'`

**Gejala:** `zookeeper-shell` gagal SASL karena `KAFKA_OPTS` menunjuk ke `zk_server_jaas.conf` yang tidak punya section `Client`.

**Solusi:** Buat file JAAS terpisah (`zk_client_jaas.conf`) dengan section `Client`. Gunakan file ini saat menjalankan `zookeeper-shell`.

### Issue 5: `handshake_failure` pada Secure Port

**Penyebab:** ZK hanya punya `ssl.quorum.keyStore.*` (untuk quorum) tapi belum punya `ssl.keyStore.*` (untuk client port).

**Solusi:** Tambahkan `ssl.keyStore.*`, `ssl.trustStore.*`, dan `ssl.clientAuth=none` di setiap `zookeeper*.properties`.

> **Key learning:** `ssl.quorum.*` = antar ZK nodes. `ssl.*` (tanpa quorum) = untuk client port. Keduanya **harus** dikonfigurasi terpisah.

---

## ✅ 2.10 Ringkasan Hasil Testing Step 2

| # | Test | Port | Hasil | Keterangan |
|---|------|------|-------|------------|
| 1 | Quorum SSL + Leader Election | 2888/3888 | ✅ Berhasil | TLS handshake berhasil, leader election normal |
| 2 | Plain tanpa SASL | 2181 | ❌ Ditolak | `AuthFailed` — SASL enforcement aktif |
| 3 | Plain dengan SASL | 2181 | ✅ Berhasil | `SaslAuthenticated` + `ls /` berhasil |
| 4 | SSL verify (openssl) | 2281 | ✅ Berhasil | `Verify return code: 0 (ok)` |
| 5 | ZK Shell via SSL dengan SASL | 2281 | ✅ Berhasil | `SaslAuthenticated` + `ls /` berhasil |

### Security yang Tercapai di Step 2:

```
✅ Quorum Encryption    → Komunikasi antar ZK nodes terenkripsi (TLSv1.2)
✅ Quorum Authentication → ZK nodes saling autentikasi via SASL Digest
✅ Client Authentication → Hanya client dengan credential SASL yang bisa akses ZK
✅ Secure Client Port    → Port 2281-2283 serve SSL (terverifikasi via openssl)
✅ Client SSL Connection dengan SASL → Hanya client dengan credential SASL yang bisa akses ZK
```

### File yang Dibuat/Dimodifikasi di Step 2:

| File | Lokasi | Fungsi |
|------|--------|--------|
| `ca-key`, `ca-cert` | `/etc/kafka/secrets/` | CA private key & certificate |
| `zk[1-3].keystore.jks` | `/etc/kafka/secrets/` | Keystore per ZK node |
| `zk.truststore.jks` | `/etc/kafka/secrets/` | Truststore shared |
| `zk_server_jaas.conf` | `/etc/kafka/` | JAAS untuk ZK server |
| `zk_client_jaas.conf` | `/etc/kafka/` | JAAS untuk ZK client testing |
| `zookeeper[1-3].properties` | `/etc/kafka/` | Config ZK nodes (updated) |

---

# 🔗 3️⃣ ZOOKEEPER CLIENT & KAFKA↔ZK CONNECTION SECURITY

## Mengapa Step Ini Diperlukan?

Setelah ZooKeeper quorum di-secure, langkah selanjutnya adalah mengamankan **koneksi dari Kafka Broker ke ZooKeeper**. Saat ini Kafka masih connect ke ZK via PLAINTEXT — artinya:

- Metadata yang dikirim broker ke ZK (topic config, partition assignment) masih plaintext.
- Siapa saja bisa connect ke ZK port dan membaca/menulis data.
- Broker tidak membuktikan identitasnya ke ZK.

## 3.1 Generate Broker Keystore untuk ZK Connection

```bash
cd /etc/kafka/secrets

# Generate keystore untuk broker (koneksi ke ZK)
sudo keytool -keystore kafka.zk.keystore.jks \
  -alias broker-zk \
  -validity 365 \
  -genkey -keyalg RSA \
  -dname "CN=localhost,O=Lab,L=Jakarta,ST=DKI,C=ID" \
  -storepass password \
  -keypass password

# Create CSR
sudo keytool -keystore kafka.zk.keystore.jks \
  -alias broker-zk \
  -certreq -file broker-zk.csr \
  -storepass password

# Sign dengan CA
sudo openssl x509 -req \
  -CA ca-cert -CAkey ca-key \
  -in broker-zk.csr \
  -out broker-zk-signed.crt \
  -days 365 -CAcreateserial

# Import CA cert ke keystore
sudo keytool -keystore kafka.zk.keystore.jks \
  -alias CARoot -import -file ca-cert \
  -storepass password -noprompt

# Import signed cert ke keystore
sudo keytool -keystore kafka.zk.keystore.jks \
  -alias broker-zk -import -file broker-zk-signed.crt \
  -storepass password -noprompt
```

## 3.2 Buat JAAS File untuk Kafka (sebagai ZK Client)

Buat file `/etc/kafka/kafka_zk_client_jaas.conf`:

```properties
Client {
  org.apache.zookeeper.server.auth.DigestLoginModule required
  username="zkadmin"
  password="zkadmin-secret";
};
```

> **Section `Client`** digunakan oleh Kafka ketika connect ke ZooKeeper sebagai client.

## 3.3 Update server.properties (ZooKeeper Connection)

Tambahkan ke `server.properties`:

```properties
# === ZooKeeper TLS Connection ===
zookeeper.ssl.client.enable=true
zookeeper.clientCnxnSocket=org.apache.zookeeper.ClientCnxnSocketNetty
zookeeper.ssl.protocol=TLSv1.2

zookeeper.ssl.keystore.location=/etc/kafka/secrets/kafka.zk.keystore.jks
zookeeper.ssl.keystore.password=password

zookeeper.ssl.truststore.location=/etc/kafka/secrets/zk.truststore.jks
zookeeper.ssl.truststore.password=password

# Update zookeeper.connect jika menggunakan secure port
# zookeeper.connect=localhost:2181,localhost:2182,localhost:2183
zookeeper.connect=localhost:2281,localhost:2282,localhost:2283
```

## 3.4 Set Environment Variable untuk confluent-server (systemd)

### 3.4.1. Buat override file
Jalankan:
```
sudo systemctl edit confluent-server

# Ini akan membuka editor dan membuat file:
/etc/systemd/system/confluent-server.service.d/override.conf
```
isi dengan:
```bash
[Service]
Environment="KAFKA_OPTS=-Djava.security.auth.login.config=/etc/kafka/kafka_zk_client_jaas.conf"
```
<img width="1019" height="115" alt="image" src="https://github.com/user-attachments/assets/867ce597-b00d-4916-bfe0-6ed943c38b16" />

Save dan keluar.

### 3.4.2. Reload systemd
```
sudo systemctl daemon-reload
```

## 3.5 Restart Kafka Broker

```bash
sudo systemctl restart confluent-server
```

### 3.5.1. Verifikasi JAAS Ter-load
Cek dengan:
```
ps aux | grep java | grep kafka_zk_client_jaas.conf
```
Harus muncul:
```
-Djava.security.auth.login.config=/etc/kafka/kafka_zk_client_jaas.conf
```
<img width="928" height="128" alt="image" src="https://github.com/user-attachments/assets/adefee5a-b449-470f-804c-04d0c7293d21" />

Kalau tidak muncul → JAAS tidak ter-load.

---

## 🧪 3.6 Testing ZooKeeper Client & Kafka↔ZK Connection

### Test 1 — Verifikasi Broker Connect ke ZK via SSL

```bash
grep "zookeeper.ssl.client.enable\|zookeeper.clientCnxnSocket\|zookeeper.connect" /etc/kafka/server.properties
```
- Broker dikonfigurasi untuk connect via SSL (zookeeper.ssl.client.enable=true, port 2281)
  
<img width="1203" height="176" alt="image" src="https://github.com/user-attachments/assets/721c2d7b-779e-40d4-8c93-24d3175b62fc" />

### Test 2 — Broker Registered di ZK

```bash
# Gunakan ZK shell dengan SSL config
KAFKA_OPTS="
-Djava.security.auth.login.config=/etc/kafka/zk_client_jaas.conf
-Dzookeeper.client.secure=true
-Dzookeeper.clientCnxnSocket=org.apache.zookeeper.ClientCnxnSocketNetty
-Dzookeeper.ssl.trustStore.location=/etc/kafka/secrets/zk.truststore.jks
-Dzookeeper.ssl.trustStore.password=password
" zookeeper-shell localhost:2281 ls /brokers/ids
```

**Expected:**

```
[0]   # atau ID broker Anda
```
<img width="1174" height="374" alt="image" src="https://github.com/user-attachments/assets/221d5c7e-5512-43cb-b11a-2461cebacf9a" />

- Broker berhasil register ke ZK yang hanya bisa diakses via SSL port → berarti koneksi SSL berhasil

### Test 3 — Kafka Masih Bisa Membuat Topic

```bash
kafka-topics --create \
  --bootstrap-server localhost:9092 \
  --topic zk-test-topic \
  --partitions 1 \
  --replication-factor 1
```

**Expected:**

```
Created topic zk-test-topic.
```
<img width="753" height="140" alt="image" src="https://github.com/user-attachments/assets/14ebd1d0-b077-4cfa-acbd-b9994fd8d183" />


### Test 4 — Non-SSL ZK Client Ditolak

```bash
# Coba connect ZK shell tanpa SSL config
zookeeper-shell localhost:2181 <<< "ls /"
```

**Expected:** Koneksi gagal atau timeout karena ZK sekarang require SSL.
<img width="1857" height="354" alt="image" src="https://github.com/user-attachments/assets/c04ff8d9-f842-4d48-b239-53a912acc2c5" />

### Test 5 — Verifikasi Metadata via ZK

```bash
KAFKA_OPTS="
-Djava.security.auth.login.config=/etc/kafka/zk_client_jaas.conf
-Dzookeeper.client.secure=true
-Dzookeeper.clientCnxnSocket=org.apache.zookeeper.ClientCnxnSocketNetty
-Dzookeeper.ssl.trustStore.location=/etc/kafka/secrets/zk.truststore.jks
-Dzookeeper.ssl.trustStore.password=password
" zookeeper-shell localhost:2281 get /brokers/ids/0
```
<img width="1866" height="415" alt="image" src="https://github.com/user-attachments/assets/c25a45b6-e208-42a8-a221-7e0be99e3c37" />

**Expected:** Menampilkan metadata broker (host, port, endpoints).

> **Kesimpulan Test:** Koneksi broker → ZK via SSL **berhasil** (terbukti dari `SaslAuthenticated` di port 2281). Listener masih PLAINTEXT karena Step 4 belum dikerjakan — akan berubah ke SASL_SSL setelah Inter-Broker Security dikonfigurasi.
---

# 🔄 4️⃣ KAFKA INTER-BROKER SECURITY (Encryption & Authentication)

## Mengapa Inter-Broker Perlu Secure?

Kafka broker berkomunikasi satu sama lain untuk:

- **Replication** — follower replica fetch data dari leader partition
- **Controller communication** — controller mengirim perintah ke broker lain
- **Group coordination** — koordinasi consumer group rebalance

Jika inter-broker communication tidak secure:
- Data replication bisa di-sniff (data terekspos)
- Rogue broker bisa join cluster
- Man-in-the-middle attack pada replication

> **Catatan:** Pada lab ini hanya 1 broker, tapi konfigurasi ini penting untuk cluster multi-broker dan harus disetup sekarang agar siap scale.

## 4.1 Generate Broker Keystore & Truststore

```bash
cd /etc/kafka/secrets

# Generate Broker Keystore
sudo keytool -keystore kafka.server.keystore.jks \
  -alias broker \
  -validity 365 \
  -genkey -keyalg RSA \
  -dname "CN=localhost,O=Lab,L=Jakarta,ST=DKI,C=ID" \
  -storepass password \
  -keypass password

# Create CSR
sudo keytool -keystore kafka.server.keystore.jks \
  -alias broker \
  -certreq -file broker.csr \
  -storepass password

# Sign with CA
sudo openssl x509 -req \
  -CA ca-cert -CAkey ca-key \
  -in broker.csr \
  -out broker-signed.crt \
  -days 365 -CAcreateserial

# Create Truststore
sudo keytool -keystore kafka.server.truststore.jks \
  -alias CARoot -import -file ca-cert \
  -storepass password -noprompt

# Import CA cert ke keystore
sudo keytool -keystore kafka.server.keystore.jks \
  -alias CARoot -import -file ca-cert \
  -storepass password -noprompt

# Import signed cert ke keystore
sudo keytool -keystore kafka.server.keystore.jks \
  -alias broker -import -file broker-signed.crt \
  -storepass password -noprompt
```

**Verifikasi:**

```bash
keytool -list -v -keystore kafka.server.keystore.jks -storepass password | grep -A2 "Alias"
```
<img width="1363" height="192" alt="image" src="https://github.com/user-attachments/assets/5296aaf6-1ce8-4f64-8e60-7db201c60262" />

## 4.2 Buat JAAS File untuk Kafka Broker

Buat file `/etc/kafka/kafka_server_jaas.conf`:

```properties
KafkaServer {
  org.apache.kafka.common.security.plain.PlainLoginModule required
  username="admin"
  password="admin-secret"
  user_admin="admin-secret"
  user_user1="user1-secret"
  user_user2="user2-secret";
};

Client {
  org.apache.zookeeper.server.auth.DigestLoginModule required
  username="zkadmin"
  password="zkadmin-secret";
};
```

**Penjelasan detail:**

| Property | Fungsi |
|----------|--------|
| `username="admin"` | Username yang digunakan broker ini untuk inter-broker authentication |
| `password="admin-secret"` | Password broker ini untuk inter-broker authentication |
| `user_admin="admin-secret"` | Mendefinisikan user "admin" dengan password "admin-secret" |
| `user_user1="user1-secret"` | Mendefinisikan user "user1" — akan digunakan oleh client |
| `user_user2="user2-secret"` | Mendefinisikan user "user2" — akan digunakan oleh client |
| Section `Client` | Credential untuk connect ke ZooKeeper (dari step sebelumnya) |

> **Penting:** `username` dan `password` di level KafkaServer adalah credential yang digunakan broker untuk berkomunikasi dengan broker lain. Format `user_<username>="<password>"` mendefinisikan user database.

## 4.3 Update server.properties (Inter-Broker)

```properties
# === Listeners ===
# Port 9092 tetap PLAINTEXT untuk sementara (akan dihapus di step berikutnya)
# Port 9093 untuk SSL
# Port 9094 untuk SASL_SSL (inter-broker dan client)
listeners=PLAINTEXT://:9092,SSL://:9093,SASL_SSL://:9094
advertised.listeners=PLAINTEXT://localhost:9092,SSL://localhost:9093,SASL_SSL://localhost:9094

# === Inter-Broker Protocol ===
security.inter.broker.protocol=SASL_SSL
sasl.mechanism.inter.broker.protocol=PLAIN

# === SSL Configuration ===
ssl.keystore.location=/etc/kafka/secrets/kafka.server.keystore.jks
ssl.keystore.password=password
ssl.key.password=password

ssl.truststore.location=/etc/kafka/secrets/kafka.server.truststore.jks
ssl.truststore.password=password

# === SASL Configuration ===
sasl.enabled.mechanisms=PLAIN
```
<img width="979" height="710" alt="image" src="https://github.com/user-attachments/assets/51cff89b-2a6f-4cb8-bb3d-b9aac46b3d34" />

## 4.4 Update Environment Variable

Edit `override file untuk confluent-server`:

```bash
sudo systemctl edit confluent-server

# replace Environment="KAFKA_OPTS=-Djava.security.auth.login.config=/etc/kafka/kafka_zk_client_jaas.conf"
# dengan

Environment="KAFKA_OPTS=-Djava.security.auth.login.config=/etc/kafka/kafka_server_jaas.conf"
```
Save dan keluar.

<img width="683" height="270" alt="image" src="https://github.com/user-attachments/assets/f8e9c48e-e7cf-4613-a3f8-b4a70e4bdca9" />

### 4.4.1. Reload systemd
```
sudo systemctl daemon-reload
```

> **Catatan:** File JAAS ini sekarang berisi section `KafkaServer` (untuk broker) DAN section `Client` (untuk koneksi ke ZooKeeper).

## 4.5 Restart Kafka Broker

```bash
sudo systemctl restart confluent-server
```

---

## 🧪 4.6 Testing Kafka Inter-Broker Security

### Test 1 — Verifikasi Broker Start dengan SASL_SSL

```
sudo journalctl -u confluent-server | grep -i "started\|registered broker\|sasl_ssl" | tail -10
```

**Actual result:**
```
security.protocol = SASL_SSL
INFO [KafkaServer id=0] started (kafka.server.KafkaServer)
```
<img width="1855" height="260" alt="image" src="https://github.com/user-attachments/assets/9b277ac9-b742-4bd9-90ff-e5802d27e94c" />

> **Penjelasan:** `security.protocol = SASL_SSL` membuktikan inter-broker protocol sudah dikonfigurasi SASL_SSL.
> `[KafkaServer id=0] started` membuktikan broker berhasil start dengan konfigurasi tersebut. ✅

### Test 2 — Verifikasi Port Listening

```bash
sudo ss -tlnp | grep -E "9092|9093|9094"
```

**Expected:**

```
LISTEN  0  50  *:9092  *:*  users:(("java",...))
LISTEN  0  50  *:9093  *:*  users:(("java",...))
LISTEN  0  50  *:9094  *:*  users:(("java",...))
```
<img width="1025" height="136" alt="image" src="https://github.com/user-attachments/assets/46b0ef10-2076-4cb9-8b7a-b57365787b03" />


### Test 3 — SSL Handshake Test

```bash
openssl s_client -connect localhost:9093 -tls1_2 </dev/null 2>/dev/null | head -20
```
<img width="1299" height="438" alt="image" src="https://github.com/user-attachments/assets/d0f45ca7-cb40-488d-8794-04a9a4bbf3f1" />

**Expected:** Menampilkan certificate chain dan "SSL handshake has read ... bytes".

### Test 4 — Verifikasi SASL_SSL Port

```bash
openssl s_client -connect localhost:9094 -tls1_2 </dev/null 2>/dev/null | head -5
```
<img width="1258" height="145" alt="image" src="https://github.com/user-attachments/assets/fb867ef5-b1a0-4199-bd28-5f0b45fb537b" />

**Expected:** SSL connection established (SASL authentication belum dilakukan, tapi SSL layer berjalan).

### Test 5 — Kafka Metadata Request via PLAINTEXT (Masih Bekerja)

```bash
kafka-broker-api-versions --bootstrap-server localhost:9092
```
<img width="1061" height="198" alt="image" src="https://github.com/user-attachments/assets/46b179d8-07d2-4348-8e86-44e889113a78" />

**Expected:** Menampilkan daftar API versions. PLAINTEXT masih aktif karena belum dihapus.

---

# 🖥️ 5️⃣ KAFKA CLIENT SECURITY (Encryption & Authentication)

## Mengapa Client Perlu Secure?

Setelah inter-broker secure, langkah selanjutnya adalah mengamankan koneksi **producer dan consumer ke broker**. Ini memastikan:

- Data yang dikirim producer ke broker terenkripsi.
- Data yang diterima consumer dari broker terenkripsi.
- Hanya client yang terautentikasi yang bisa mengakses cluster.
- Setiap client memiliki identitas unik untuk authorization (ACL) nanti.

## 5.1 Buat Client Properties File

### File: `client-ssl.properties` (SSL Only)

```properties
security.protocol=SSL
ssl.truststore.location=/etc/kafka/secrets/kafka.server.truststore.jks
ssl.truststore.password=password
```

### File: `client-sasl-ssl.properties` (SASL + SSL — Recommended)

```properties
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
ssl.truststore.location=/etc/kafka/secrets/kafka.server.truststore.jks
ssl.truststore.password=password

sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="user1" \
  password="user1-secret";
```

### File: `admin-sasl-ssl.properties` (Admin User)

```properties
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
ssl.truststore.location=/etc/kafka/secrets/kafka.server.truststore.jks
ssl.truststore.password=password

sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="admin" \
  password="admin-secret";
```

## 5.2 (Opsional) Hapus PLAINTEXT Listener

Setelah semua komponen sudah menggunakan SSL/SASL_SSL, hapus PLAINTEXT listener untuk keamanan penuh.

Update `server.properties`:

```properties
# Hapus PLAINTEXT, hanya sisakan SSL dan SASL_SSL
listeners=SSL://:9093,SASL_SSL://:9094
advertised.listeners=SSL://localhost:9093,SASL_SSL://localhost:9094
```

Restart broker:

```bash
sudo systemctl restart confluent-server
```

> **Peringatan:** Setelah PLAINTEXT dihapus, semua tool CLI harus menggunakan `--command-config` dengan properties file yang sesuai.

## 5.3 Update Schema Registry & Control Center (Opsional)

Jika menggunakan Schema Registry dan Control Center, update konfigurasi mereka juga:

### Schema Registry (`schema-registry.properties`)

```properties
kafkastore.bootstrap.servers=SASL_SSL://localhost:9094
kafkastore.security.protocol=SASL_SSL
kafkastore.sasl.mechanism=PLAIN
kafkastore.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="admin" \
  password="admin-secret";
kafkastore.ssl.truststore.location=/etc/kafka/secrets/kafka.server.truststore.jks
kafkastore.ssl.truststore.password=password
```

### Control Center (`control-center.properties`)

```properties
bootstrap.servers=SASL_SSL://localhost:9094
confluent.controlcenter.streams.security.protocol=SASL_SSL
confluent.controlcenter.streams.sasl.mechanism=PLAIN
confluent.controlcenter.streams.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="admin" \
  password="admin-secret";
confluent.controlcenter.streams.ssl.truststore.location=/etc/kafka/secrets/kafka.server.truststore.jks
confluent.controlcenter.streams.ssl.truststore.password=password
```

---

## 🧪 5.4 Testing Kafka Client Security

### Test 1 — SSL Tanpa Config (Harus Gagal)

```bash
kafka-console-producer \
  --bootstrap-server localhost:9093 \
  --topic test-topic
```

**Expected:**

```
ERROR: SSL handshake failed
org.apache.kafka.common.errors.SslAuthenticationException
```

**Penjelasan:** Client mencoba connect ke SSL port tanpa truststore, sehingga tidak bisa verifikasi certificate broker.

### Test 2 — SSL Dengan Config (Harus Berhasil)

```bash
kafka-console-producer \
  --bootstrap-server localhost:9093 \
  --topic test-topic \
  --producer.config client-ssl.properties
```

**Expected:** Producer prompt muncul, bisa mengetik pesan.

```
>Hello SSL
>Test message
>
```

### Test 3 — SASL_SSL Tanpa Credential (Harus Gagal)

```bash
kafka-console-producer \
  --bootstrap-server localhost:9094 \
  --topic test-topic
```

**Expected:**

```
ERROR: SASL authentication failed
org.apache.kafka.common.errors.SaslAuthenticationException: Authentication failed
```

**Penjelasan:** Client tidak menyediakan username/password untuk SASL authentication.

### Test 4 — SASL_SSL Dengan Credential Salah (Harus Gagal)

Buat file `client-wrong.properties`:

```properties
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
ssl.truststore.location=/etc/kafka/secrets/kafka.server.truststore.jks
ssl.truststore.password=password

sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="user1" \
  password="wrong-password";
```

```bash
kafka-console-producer \
  --bootstrap-server localhost:9094 \
  --topic test-topic \
  --producer.config client-wrong.properties
```

**Expected:**

```
ERROR: Authentication failed: Invalid username or password
```

### Test 5 — SASL_SSL Dengan Credential Benar (Harus Berhasil)

```bash
kafka-console-producer \
  --bootstrap-server localhost:9094 \
  --topic test-topic \
  --producer.config client-sasl-ssl.properties
```

**Expected:** Producer prompt muncul.

```
>Hello SASL_SSL
>Authenticated message
>
```

### Test 6 — Consume via SASL_SSL

```bash
kafka-console-consumer \
  --bootstrap-server localhost:9094 \
  --topic test-topic \
  --from-beginning \
  --consumer.config client-sasl-ssl.properties
```

**Expected:** Pesan yang diproduce sebelumnya muncul.

### Test 7 — Verifikasi User yang Berbeda

```bash
# Buat properties untuk user2
cat > client-user2.properties << EOF
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
ssl.truststore.location=/etc/kafka/secrets/kafka.server.truststore.jks
ssl.truststore.password=password

sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="user2" \
  password="user2-secret";
EOF

# Test produce sebagai user2
kafka-console-producer \
  --bootstrap-server localhost:9094 \
  --topic test-topic \
  --producer.config client-user2.properties
```

**Expected:** Berhasil — user2 juga terdaftar di JAAS file.

---

# 🛂 6️⃣ ENABLE ACL AUTHORIZATION

## Mengapa ACL Diperlukan?

Saat ini, meskipun sudah ada authentication, **semua user yang terautentikasi memiliki akses penuh** ke semua resource. user1 bisa baca semua topic, user2 bisa delete topic production, dst.

ACL memungkinkan kontrol granular:
- user1 hanya boleh **write** ke topic `orders`
- user2 hanya boleh **read** dari topic `orders`
- Tidak ada yang boleh **delete topic** kecuali admin

## 6.1 Enable Authorizer di server.properties

Tambahkan:

```properties
# === ACL Authorization ===
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
super.users=User:admin
allow.everyone.if.no.acl.found=false
```

**Penjelasan:**

| Property | Fungsi |
|----------|--------|
| `authorizer.class.name` | Mengaktifkan ACL authorizer bawaan Kafka |
| `super.users=User:admin` | User "admin" bypass semua ACL — seperti root di Linux |
| `allow.everyone.if.no.acl.found=false` | Jika tidak ada ACL untuk resource, **DENY semua** (secure by default) |

> **Peringatan:** Setelah `allow.everyone.if.no.acl.found=false` diaktifkan, semua operasi non-admin akan gagal sampai ACL ditambahkan. Pastikan admin properties file sudah siap.

Restart broker:

```bash
sudo systemctl restart confluent-server
```

## 6.2 Buat Topic untuk Testing ACL

```bash
kafka-topics --create \
  --bootstrap-server localhost:9094 \
  --topic secure-topic \
  --partitions 3 \
  --replication-factor 1 \
  --command-config admin-sasl-ssl.properties
```

> **Penting:** Gunakan `admin-sasl-ssl.properties` karena sekarang hanya super user yang bisa create topic.

---

## 🧪 6.3 FULL ACL TEST MATRIX

### Test A — No ACL (user1 produce → Harus Gagal)

```bash
kafka-console-producer \
  --bootstrap-server localhost:9094 \
  --topic secure-topic \
  --producer.config client-sasl-ssl.properties
```

**Expected:**

```
ERROR: Topic authorization failed.
org.apache.kafka.common.errors.TopicAuthorizationException: 
  Not authorized to access topics: [secure-topic]
```

**Penjelasan:** user1 terautentikasi, tapi tidak ada ACL yang mengizinkan akses ke `secure-topic`.

---

### Test B — Add Write ACL (user1 produce → Harus Berhasil)

```bash
kafka-acls --bootstrap-server localhost:9094 \
  --command-config admin-sasl-ssl.properties \
  --add --allow-principal User:user1 \
  --operation Write --topic secure-topic
```

Verifikasi ACL:

```bash
kafka-acls --bootstrap-server localhost:9094 \
  --command-config admin-sasl-ssl.properties \
  --list --topic secure-topic
```

**Expected output:**

```
Current ACLs for resource `ResourcePattern(resourceType=TOPIC, name=secure-topic, patternType=LITERAL)`:
  (principal=User:user1, host=*, operation=WRITE, permissionType=ALLOW)
```

Test produce:

```bash
kafka-console-producer \
  --bootstrap-server localhost:9094 \
  --topic secure-topic \
  --producer.config client-sasl-ssl.properties
```

**Expected:** Berhasil mengirim pesan.

```
>Pesan pertama dari user1
>Pesan kedua dari user1
>
```

Test consume (Harus Gagal):

```bash
kafka-console-consumer \
  --bootstrap-server localhost:9094 \
  --topic secure-topic \
  --from-beginning \
  --group test-group \
  --consumer.config client-sasl-ssl.properties
```

**Expected:**

```
ERROR: Topic authorization failed (Read operation not allowed)
```

**Penjelasan:** user1 hanya punya Write ACL, belum ada Read.

---

### Test C — Add Read + Group ACL (user1 consume → Harus Berhasil)

```bash
# Add Read untuk topic
kafka-acls --bootstrap-server localhost:9094 \
  --command-config admin-sasl-ssl.properties \
  --add --allow-principal User:user1 \
  --operation Read --topic secure-topic

# Add Read untuk consumer group
kafka-acls --bootstrap-server localhost:9094 \
  --command-config admin-sasl-ssl.properties \
  --add --allow-principal User:user1 \
  --operation Read --group test-group
```

> **Penting:** Consumer memerlukan DUA ACL — Read pada topic DAN Read pada consumer group. Tanpa group ACL, consumer tidak bisa join group.

Test consume:

```bash
kafka-console-consumer \
  --bootstrap-server localhost:9094 \
  --topic secure-topic \
  --from-beginning \
  --group test-group \
  --consumer.config client-sasl-ssl.properties
```

**Expected:** Berhasil membaca pesan yang sudah diproduce.

```
Pesan pertama dari user1
Pesan kedua dari user1
```

---

### Test D — Explicit Deny (Override Allow)

```bash
kafka-acls --bootstrap-server localhost:9094 \
  --command-config admin-sasl-ssl.properties \
  --add --deny-principal User:user1 \
  --operation Write --topic secure-topic
```

Verifikasi ACL:

```bash
kafka-acls --bootstrap-server localhost:9094 \
  --command-config admin-sasl-ssl.properties \
  --list --topic secure-topic
```

**Expected output:**

```
(principal=User:user1, host=*, operation=WRITE, permissionType=ALLOW)
(principal=User:user1, host=*, operation=WRITE, permissionType=DENY)
(principal=User:user1, host=*, operation=READ, permissionType=ALLOW)
```

Test produce:

```bash
kafka-console-producer \
  --bootstrap-server localhost:9094 \
  --topic secure-topic \
  --producer.config client-sasl-ssl.properties
```

**Expected:**

```
ERROR: Topic authorization failed.
```

**Penjelasan:** DENY selalu override ALLOW. Meskipun ada ALLOW WRITE, DENY WRITE menang.

Cleanup — hapus deny agar test selanjutnya berjalan:

```bash
kafka-acls --bootstrap-server localhost:9094 \
  --command-config admin-sasl-ssl.properties \
  --remove --deny-principal User:user1 \
  --operation Write --topic secure-topic
```

---

### Test E — Wildcard Topic ACL

```bash
kafka-acls --bootstrap-server localhost:9094 \
  --command-config admin-sasl-ssl.properties \
  --add --allow-principal User:user2 \
  --operation Read --operation Write --topic '*'
```

Test:

```bash
# user2 produce ke topic apapun
kafka-console-producer \
  --bootstrap-server localhost:9094 \
  --topic secure-topic \
  --producer.config client-user2.properties

# user2 produce ke topic lain
kafka-console-producer \
  --bootstrap-server localhost:9094 \
  --topic another-topic \
  --producer.config client-user2.properties
```

**Expected:** Kedua produce berhasil karena wildcard `'*'` mencakup semua topic.

> **Peringatan:** Wildcard ACL sangat broad — gunakan dengan hati-hati di production. Lebih baik gunakan prefixed ACL (e.g., `--resource-pattern-type prefixed --topic "team-a-"`) untuk scope yang lebih terkontrol.

---

### Test F — Cluster Level ACL (Create Topic)

```bash
# user1 coba create topic tanpa Cluster ACL
kafka-topics --create \
  --bootstrap-server localhost:9094 \
  --topic user1-topic \
  --partitions 1 \
  --replication-factor 1 \
  --command-config client-sasl-ssl.properties
```

**Expected:**

```
ERROR: Cluster authorization failed.
```

```bash
# Tambahkan Cluster Create ACL
kafka-acls --bootstrap-server localhost:9094 \
  --command-config admin-sasl-ssl.properties \
  --add --allow-principal User:user1 \
  --operation Create --cluster
```

```bash
# Coba lagi
kafka-topics --create \
  --bootstrap-server localhost:9094 \
  --topic user1-topic \
  --partitions 1 \
  --replication-factor 1 \
  --command-config client-sasl-ssl.properties
```

**Expected:**

```
Created topic user1-topic.
```

---

### Test G — Admin Super User (Bypass ACL)

```bash
# Admin bisa melakukan apapun tanpa ACL
kafka-topics --list \
  --bootstrap-server localhost:9094 \
  --command-config admin-sasl-ssl.properties
```

**Expected:** Semua topic terlisting — admin bypass semua ACL.

---

### Test H — List Semua ACL

```bash
kafka-acls --bootstrap-server localhost:9094 \
  --command-config admin-sasl-ssl.properties \
  --list
```

**Expected:** Menampilkan semua ACL yang sudah dibuat.

---

# 🧪 7️⃣ FULL TESTING — PRODUCE & CONSUME WITH SECURITY

## End-to-End Test Scenario

### Skenario: Order Processing System

```
Producer (user1) → Topic: orders → Consumer (user2)
```

### Step 1: Setup ACL

```bash
# user1 hanya boleh Write ke orders
kafka-acls --bootstrap-server localhost:9094 \
  --command-config admin-sasl-ssl.properties \
  --add --allow-principal User:user1 \
  --operation Write --topic orders

# user2 hanya boleh Read dari orders + group
kafka-acls --bootstrap-server localhost:9094 \
  --command-config admin-sasl-ssl.properties \
  --add --allow-principal User:user1 \
  --operation Describe --topic orders

kafka-acls --bootstrap-server localhost:9094 \
  --command-config admin-sasl-ssl.properties \
  --add --allow-principal User:user2 \
  --operation Read --topic orders

kafka-acls --bootstrap-server localhost:9094 \
  --command-config admin-sasl-ssl.properties \
  --add --allow-principal User:user2 \
  --operation Describe --topic orders

kafka-acls --bootstrap-server localhost:9094 \
  --command-config admin-sasl-ssl.properties \
  --add --allow-principal User:user2 \
  --operation Read --group order-consumers
```

### Step 2: Create Topic (Admin)

```bash
kafka-topics --create \
  --bootstrap-server localhost:9094 \
  --topic orders \
  --partitions 3 \
  --replication-factor 1 \
  --command-config admin-sasl-ssl.properties
```

### Step 3: Start Consumer (Terminal 1)

```bash
kafka-console-consumer \
  --bootstrap-server localhost:9094 \
  --topic orders \
  --group order-consumers \
  --consumer.config client-user2.properties
```

### Step 4: Produce Messages (Terminal 2)

```bash
kafka-console-producer \
  --bootstrap-server localhost:9094 \
  --topic orders \
  --producer.config client-sasl-ssl.properties
```

Ketik:

```
{"orderId":"001","product":"Laptop","qty":1}
{"orderId":"002","product":"Mouse","qty":5}
{"orderId":"003","product":"Keyboard","qty":2}
```

### Step 5: Verifikasi

**Di Terminal 1 (consumer):**

```
{"orderId":"001","product":"Laptop","qty":1}
{"orderId":"002","product":"Mouse","qty":5}
{"orderId":"003","product":"Keyboard","qty":2}
```

### Step 6: Negative Test — user2 Coba Produce (Harus Gagal)

```bash
kafka-console-producer \
  --bootstrap-server localhost:9094 \
  --topic orders \
  --producer.config client-user2.properties
```

**Expected:**

```
ERROR: Topic authorization failed.
```

### Step 7: Negative Test — user1 Coba Consume (Harus Gagal)

```bash
kafka-console-consumer \
  --bootstrap-server localhost:9094 \
  --topic orders \
  --group test-group \
  --consumer.config client-sasl-ssl.properties
```

**Expected:**

```
ERROR: Topic authorization failed (user1 tidak punya Read ACL pada topic orders)
```

> **Catatan:** user1 hanya punya Write dan Describe. user2 hanya punya Read dan Describe. Setiap user terbatas pada operasinya masing-masing.

---

# 📝 8️⃣ VALIDATION CHECKLIST

| # | Test | Layer | Expected | Status |
|---|------|-------|----------|--------|
| 1 | ZK plain connection ditolak | ZK Quorum | ❌ Gagal | ☐ |
| 2 | ZK quorum leader election berjalan | ZK Quorum | ✅ Berhasil | ☐ |
| 3 | ZK shell dengan SSL berhasil | ZK Quorum | ✅ Berhasil | ☐ |
| 4 | Broker connect ke ZK via SSL | ZK↔Kafka | ✅ Berhasil | ☐ |
| 5 | Broker registered di ZK | ZK↔Kafka | ✅ Berhasil | ☐ |
| 6 | Non-SSL ZK client ditolak | ZK↔Kafka | ❌ Gagal | ☐ |
| 7 | Broker start dengan SASL_SSL | Inter-Broker | ✅ Berhasil | ☐ |
| 8 | SSL handshake berhasil di port 9093 | Inter-Broker | ✅ Berhasil | ☐ |
| 9 | Client SSL tanpa config gagal | Client | ❌ Gagal | ☐ |
| 10 | Client SSL dengan config berhasil | Client | ✅ Berhasil | ☐ |
| 11 | Client SASL_SSL tanpa credential gagal | Client | ❌ Gagal | ☐ |
| 12 | Client SASL_SSL credential salah gagal | Client | ❌ Gagal | ☐ |
| 13 | Client SASL_SSL credential benar berhasil | Client | ✅ Berhasil | ☐ |
| 14 | No ACL → deny | ACL | ❌ Gagal | ☐ |
| 15 | Write ACL → produce berhasil | ACL | ✅ Berhasil | ☐ |
| 16 | Write only → consume gagal | ACL | ❌ Gagal | ☐ |
| 17 | Read + Group ACL → consume berhasil | ACL | ✅ Berhasil | ☐ |
| 18 | Deny override Allow | ACL | ❌ Gagal | ☐ |
| 19 | Admin super user bypass ACL | ACL | ✅ Berhasil | ☐ |
| 20 | E2E produce + consume dengan ACL | Full | ✅ Berhasil | ☐ |

---

# 🏁 FINAL RESULT

Cluster sekarang memiliki security berlapis (defense in depth):

```
┌─────────────────────────────────────────────────┐
│                   CLIENT LAYER                   │
│          SASL_SSL (Encrypted + Auth)             │
│     Producer ←→ Broker ←→ Consumer               │
├─────────────────────────────────────────────────┤
│                AUTHORIZATION                     │
│         ACL (Granular Access Control)            │
│  User:user1 → Write:orders                      │
│  User:user2 → Read:orders                       │
├─────────────────────────────────────────────────┤
│              INTER-BROKER LAYER                  │
│         SASL_SSL (Encrypted + Auth)              │
│        Broker ←→ Broker (Replication)            │
├─────────────────────────────────────────────────┤
│            KAFKA ↔ ZOOKEEPER LAYER               │
│         SSL + SASL (Encrypted + Auth)            │
│            Broker ←→ ZooKeeper                   │
├─────────────────────────────────────────────────┤
│             ZOOKEEPER QUORUM LAYER               │
│         SSL + SASL (Encrypted + Auth)            │
│          ZK Node ←→ ZK Node ←→ ZK Node          │
└─────────────────────────────────────────────────┘
```

**Security Properties Tercapai:**

| Property | Status | Detail |
|----------|--------|--------|
| Encrypted (Confidentiality) | ✅ | Semua traffic menggunakan TLS |
| Authenticated (Identity) | ✅ | SASL memverifikasi identitas |
| Authorized (Access Control) | ✅ | ACL mengontrol akses per resource |
| ZK Quorum Secure | ✅ | Komunikasi antar ZK node terenkripsi |
| ZK Client Secure | ✅ | Koneksi broker ke ZK terenkripsi + auth |

---

# 📋 RINGKASAN FILE & KONFIGURASI

## File Certificate & Keystore

| File | Lokasi | Fungsi |
|------|--------|--------|
| `ca-key` | `/etc/kafka/secrets/` | Private key CA |
| `ca-cert` | `/etc/kafka/secrets/` | Certificate CA |
| `zk1.keystore.jks` | `/etc/kafka/secrets/` | Keystore ZK Node 1 |
| `zk2.keystore.jks` | `/etc/kafka/secrets/` | Keystore ZK Node 2 |
| `zk3.keystore.jks` | `/etc/kafka/secrets/` | Keystore ZK Node 3 |
| `zk.truststore.jks` | `/etc/kafka/secrets/` | Truststore ZK (shared) |
| `kafka.zk.keystore.jks` | `/etc/kafka/secrets/` | Keystore Broker→ZK connection |
| `kafka.server.keystore.jks` | `/etc/kafka/secrets/` | Keystore Broker |
| `kafka.server.truststore.jks` | `/etc/kafka/secrets/` | Truststore Broker |

## File Konfigurasi

| File | Lokasi | Fungsi |
|------|--------|--------|
| `zk_server_jaas.conf` | `/etc/kafka/` | JAAS untuk ZooKeeper server |
| `kafka_server_jaas.conf` | `/etc/kafka/` | JAAS untuk Kafka broker (termasuk ZK client) |
| `zookeeper1.properties` | `/etc/kafka/` | Config ZK Node 1 |
| `zookeeper2.properties` | `/etc/kafka/` | Config ZK Node 2 |
| `zookeeper3.properties` | `/etc/kafka/` | Config ZK Node 3 |
| `server.properties` | `/etc/kafka/` | Config Kafka Broker |

## File Client Properties

| File | User | Protocol |
|------|------|----------|
| `client-ssl.properties` | - | SSL only |
| `client-sasl-ssl.properties` | user1 | SASL_SSL |
| `client-user2.properties` | user2 | SASL_SSL |
| `admin-sasl-ssl.properties` | admin | SASL_SSL |
| `zk-client-ssl.properties` | - | ZK TLS client |

---

# 🔧 TROUBLESHOOTING

## Common Issues

### 1. Broker Gagal Start Setelah Enable Security

**Cek log:**

```bash
sudo journalctl -u confluent-server -n 50 --no-pager
sudo tail -100 /var/log/kafka/server.log
```

**Common cause:** Path keystore salah, password salah, atau JAAS file tidak ditemukan.

### 2. SSL Handshake Failed

**Cek:**

```bash
# Verifikasi certificate valid
keytool -list -v -keystore kafka.server.keystore.jks -storepass password
openssl s_client -connect localhost:9093 -tls1_2
```

**Common cause:** Certificate expired, CN mismatch, atau CA cert belum di-import ke truststore.

### 3. SASL Authentication Failed

**Cek:**

```bash
# Verifikasi JAAS file loaded
ps aux | grep java.security.auth.login.config
```

**Common cause:** JAAS file path salah di `KAFKA_OPTS`, atau username/password tidak match.

### 4. ACL Authorization Failed (Padahal ACL Sudah Ditambah)

```bash
# List semua ACL
kafka-acls --bootstrap-server localhost:9094 \
  --command-config admin-sasl-ssl.properties \
  --list
```

**Common cause:** Consumer group ACL belum ditambahkan, atau principal name salah (case-sensitive).

### 5. ZooKeeper Connection Lost Setelah Enable Security

**Cek:**

```bash
sudo grep -i "error\|exception\|ssl\|auth" /var/log/kafka/zookeeper.log | tail -20
```

**Common cause:** Restart ZK sekaligus (harus rolling), atau JAAS `Client` section missing di `kafka_server_jaas.conf`.

---

# 📚 Referensi Dokumentasi Resmi

- [ZooKeeper Administration](https://zookeeper.apache.org/doc/current/zookeeperAdmin.html)
- [Kafka Security Overview](https://kafka.apache.org/documentation/#security)
- [Confluent Platform Security](https://docs.confluent.io/platform/7.9/security/overview.html)
- [Kafka Authorization (ACLs)](https://kafka.apache.org/documentation/#security_authz)
- [Confluent ACL Overview](https://docs.confluent.io/platform/7.9/security/authorization/acls/overview.html)
- [Confluent SASL Overview](https://docs.confluent.io/platform/7.9/security/authentication/sasl/overview.html)
