# Task 3: Deploy Confluent Platform dengan Security Enabled

## Automate Confluent Deployment dengan Ansible — Secure Cluster

> **Referensi Resmi:**
> [https://docs.confluent.io/ansible/current/ansible-security.html](https://docs.confluent.io/ansible/current/ansible-security.html)

> **Prasyarat:**
> Task 2 sudah selesai dan cluster tanpa security sudah berjalan dengan baik.

---

## Gambaran Umum

Task ini mengubah deployment sebelumnya menjadi **secure Confluent Platform cluster** dengan fitur:

| Security Layer    | Teknologi     |
| ----------------- | ------------- |
| Encryption        | TLS/SSL       |
| Authentication    | SASL/PLAIN    |
| Authorization     | Kafka ACL     |
| Secret Management | Ansible Vault |

Semua komunikasi antar komponen akan dienkripsi dan memerlukan autentikasi.

---

## Arsitektur Security

```
                ┌────────────────────────────┐
                │        Control Center       │
                │   cp-ansible :9021 (TLS)    │
                └─────────────┬───────────────┘
                              │
                      TLS + SASL Auth
                              │
┌─────────────────────────────────────────────────────────────┐
│                     Kafka Cluster                           │
│                                                             │
│   cp-node1            cp-node2            cp-node3          │
│                                                             │
│ ZooKeeper TLS      ZooKeeper TLS       ZooKeeper TLS        │
│ Kafka Broker TLS   Kafka Broker TLS    Kafka Broker TLS     │
│ Schema Registry    Schema Registry     Schema Registry      │
│ Kafka Connect      Kafka Connect       Kafka Connect        │
│ ksqlDB             ksqlDB              ksqlDB               │
│ REST Proxy         REST Proxy          REST Proxy           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Security Model

### 1. Encryption (TLS)

Semua komunikasi dienkripsi menggunakan **TLS certificate** yang di-generate otomatis oleh CP Ansible:

```
Broker ↔ Broker
Broker ↔ ZooKeeper
Client ↔ Broker
Component ↔ Component
```

> **Penting:** Jangan generate certificate manual dengan OpenSSL dan meletakkannya di `vars/security.yml` menggunakan `ssl_keystore_path` dll. CP Ansible akan otomatis generate certificate dengan CN dan SAN yang benar untuk setiap node. Certificate manual yang di-generate dengan `CN=localhost` akan menyebabkan TLS handshake gagal karena hostname tidak cocok.

### 2. Authentication

Kafka menggunakan **SASL/PLAIN**. User harus login sebelum mengakses broker.

Daftar user yang digunakan:

| User | Fungsi |
|---|---|
| `admin` | Inter-broker communication, admin tasks |
| `schema_registry` | Schema Registry → Broker |
| `kafka_connect` | Kafka Connect → Broker |
| `ksql` | ksqlDB → Broker |
| `control_center` | Control Center → Broker |
| `kafka_rest` | REST Proxy → Broker |
| `client` | External client (producer/consumer) |

### 3. Authorization

Kafka menggunakan **ACL (Access Control List)** dengan `AclAuthorizer`. Semua akses ditolak kecuali ada ACL eksplisit (`allow.everyone.if.no.acl.found: false`).

Service account internal didaftarkan sebagai **super.users** agar bisa bypass ACL check untuk komunikasi internal antar komponen:

```
super.users=User:admin;User:schema_registry;User:kafka_connect;User:ksql;User:control_center;User:kafka_rest
```

### 4. Secret Protection

Semua password disimpan menggunakan **Ansible Vault** sehingga tidak muncul plaintext di inventory atau version control.

---

## Struktur Direktori Project

```
~/confluent-ansible/
├── confluent.yml
├── inventory/
│   └── hosts.yml
├── vars/
│   ├── platform.yml
│   └── security.yml
└── vault/
    └── secrets.yml
```

> **Catatan:** Folder `certs/` dari instruksi awal tidak diperlukan karena CP Ansible generate certificate otomatis.

---

## Step 1: Buat File `vars/security.yml`

Buat file konfigurasi security. **Biarkan CP Ansible yang generate certificate** — jangan override dengan custom cert path.

```bash
nano ~/confluent-ansible/vars/security.yml
```

```yaml
---
ssl_enabled: true
sasl_protocol: plain

kafka_broker_custom_properties:
  super.users: "User:admin;User:schema_registry;User:kafka_connect;User:ksql;User:control_center;User:kafka_rest"
  authorizer.class.name: kafka.security.authorizer.AclAuthorizer
  allow.everyone.if.no.acl.found: false  
```

> **Penjelasan `super.users`:** Diperlukan agar service internal (Schema Registry, Connect, ksqlDB, dll) bisa berkomunikasi dengan broker tanpa perlu ACL satu per satu. Tanpa ini, semua service akan mendapat `TopicAuthorizationException` saat startup.

---

## Step 2: Update `vars/platform.yml`

Pastikan tidak ada konflik dengan `security.yml`. Hapus atau comment baris berikut jika ada:

```yaml
# HAPUS atau comment kedua baris ini dari platform.yml:
# ssl_enabled: false
# sasl_protocol: none
```

Karena TLS sudah aktif, update semua URL Schema Registry dari HTTP ke HTTPS:

```yaml
# ksqlDB
ksql_custom_properties:
  ksql.schema.registry.url: "https://cp-node1:8081"

# REST Proxy
kafka_rest_custom_properties:
  schema.registry.url: "https://cp-node1:8081"

# Schema Registry
schema_registry_custom_properties:
  listeners: "https://0.0.0.0:8081"
```

---

## Step 3: Buat File Secrets dengan Ansible Vault

### Buat `vault/secrets.yml`

```bash
mkdir -p ~/confluent-ansible/vault
nano ~/confluent-ansible/vault/secrets.yml
```

```yaml
kafka_admin_password: admin-secret
kafka_connect_password: connect-secret
kafka_ksql_password: ksql-secret
kafka_schema_registry_password: schema_registry-secret
kafka_control_center_password: control_center-secret
kafka_rest_password: kafka_rest-secret
kafka_client_password: client-secret
```

### Encrypt dengan Ansible Vault

```bash
ansible-vault encrypt vault/secrets.yml
```

Masukkan vault password saat diminta. Contoh: `P@ssword`

Verifikasi file sudah terenkripsi:

```bash
cat vault/secrets.yml
# Output harus berupa: $ANSIBLE_VAULT;1.1;AES256...
```

Untuk edit kembali:

```bash
ansible-vault edit vault/secrets.yml
```

---

## Step 4: Update `inventory/hosts.yml`

Tambahkan `sasl_users` untuk semua service account. Password mengacu ke variabel dari vault.

```yaml
all:
  vars:
    ansible_user: cpadmin
    ansible_become: true
    ansible_ssh_private_key_file: ~/.ssh/id_rsa

    sasl_users:
      admin:
        password: "{{ kafka_admin_password }}"
      connect:
        password: "{{ kafka_connect_password }}"
      ksql:
        password: "{{ kafka_ksql_password }}"
      schema_registry:
        password: "{{ kafka_schema_registry_password }}"
      control_center:
        password: "{{ kafka_control_center_password }}"
      kafka_rest:
        password: "{{ kafka_rest_password }}"
      client:
        password: "{{ kafka_client_password }}"

zookeeper:
  hosts:
    cp-node1:
      ansible_host: 192.168.56.11
    cp-node2:
      ansible_host: 192.168.56.12
    cp-node3:
      ansible_host: 192.168.56.13

kafka_broker:
  hosts:
    cp-node1:
      ansible_host: 192.168.56.11
    cp-node2:
      ansible_host: 192.168.56.12
    cp-node3:
      ansible_host: 192.168.56.13

schema_registry:
  hosts:
    cp-node1:
      ansible_host: 192.168.56.11
    cp-node2:
      ansible_host: 192.168.56.12
    cp-node3:
      ansible_host: 192.168.56.13

kafka_connect:
  hosts:
    cp-node1:
      ansible_host: 192.168.56.11
    cp-node2:
      ansible_host: 192.168.56.12
    cp-node3:
      ansible_host: 192.168.56.13

ksql:
  hosts:
    cp-node1:
      ansible_host: 192.168.56.11
    cp-node2:
      ansible_host: 192.168.56.12
    cp-node3:
      ansible_host: 192.168.56.13

kafka_rest:
  hosts:
    cp-node1:
      ansible_host: 192.168.56.11
    cp-node2:
      ansible_host: 192.168.56.12
    cp-node3:
      ansible_host: 192.168.56.13

control_center:
  hosts:
    cp-ansible:
      ansible_host: 192.168.56.10
```

---

## Step 5: Update `confluent.yml`

Pastikan semua variable file di-load di awal playbook, termasuk vault:

```yaml
- name: Load variables
  hosts: all
  tasks:
    - include_vars: vars/platform.yml
    - include_vars: vars/security.yml
    - include_vars: vault/secrets.yml
```

---

## Step 6: Sinkronisasi Waktu (Penting!)

Sebelum deploy, pastikan waktu sistem di semua node sinkron. Waktu yang tidak akurat akan menyebabkan `apt` gagal update karena repository timestamp dianggap invalid.

```bash
# Install dan aktifkan chrony
sudo apt install chrony -y
sudo systemctl enable chrony
sudo systemctl start chrony

# Paksa sync sekarang
sudo chronyc makestep

# Verifikasi
chronyc tracking
# Reference ID harus terisi (bukan 0.0.0.0)
```

---

## Step 7: Jalankan Deployment Secure Cluster

```bash
cd ~/confluent-ansible

ansible-playbook \
  -i inventory/hosts.yml \
  confluent.yml \
  --ask-vault-pass
```

Masukkan vault password saat diminta.

### Estimasi Waktu

| Skenario | Estimasi |
|---|---|
| Full deploy pertama kali | 45-60 menit |
| Re-deploy (node sudah terinstall) | 20-30 menit |
| Deploy hanya Control Center | 5-10 menit |

Untuk deploy hanya komponen tertentu:

```bash
# Hanya Control Center
ansible-playbook \
  -i inventory/hosts.yml \
  confluent.yml \
  --ask-vault-pass \
  --limit control_center
```

### Service Yang Akan Restart

Saat security diaktifkan, semua service akan restart secara berurutan:

```
ZooKeeper → Kafka Broker → Schema Registry → 
Kafka Connect → ksqlDB → REST Proxy → Control Center
```

---

## Troubleshooting

### Kafka Broker Gagal Start — Port Tidak Listening

**Gejala:** `ss -tlnp | grep 9091` kosong, broker crash sebelum sempat bind port.

**Penyebab paling umum:** Certificate di `/var/ssl/private/kafka_broker.keystore.jks` di-generate dengan `CN=localhost` padahal broker advertise dengan hostname `cp-node1`, `cp-node2`, `cp-node3`. TLS hostname verification gagal.

**Solusi:** Hapus konfigurasi custom cert dari `security.yml` dan biarkan CP Ansible generate otomatis:

```bash
# Hapus cert lama
ansible -i inventory/hosts.yml all -b -m file \
  -a "path=/var/ssl/private state=absent"

# Jalankan ulang — CP Ansible akan generate cert baru dengan CN yang benar
ansible-playbook -i inventory/hosts.yml confluent.yml --ask-vault-pass
```

---

### ClusterAction Denied di kafka-authorizer.log

**Gejala:**
```
Principal = User:admin is Denied Operation = ClusterAction
```

**Penyebab:** `allow.everyone.if.no.acl.found: false` aktif tapi user `admin` tidak punya ACL untuk inter-broker operation.

**Solusi:** Tambahkan `super.users` di `vars/security.yml`:

```yaml
kafka_broker_custom_properties:
  super.users: "User:admin;User:schema_registry;..."
```

---

### Schema Registry — TopicAuthorizationException

**Gejala:**
```
Failed trying to create or validate schema topic configuration
Caused by: TopicAuthorizationException: Authorization failed.
```

**Penyebab:** User `schema_registry` tidak punya izin untuk create/validate topic `_schemas`.

**Solusi:** Tambahkan `User:schema_registry` ke `super.users` di broker config.

---

### Kafka HTTP Server Timed Out (cp-node3)

**Gejala:**
```
IllegalStateException: Starting Kafka HTTP server timed out after 60000 ms
```

**Penyebab:** Broker mencoba konek ke Schema Registry via `http://localhost:8081` tapi Schema Registry sudah jalan dengan HTTPS.

**Solusi:** Update semua URL Schema Registry di `platform.yml` dari `http://` ke `https://`.

---

### apt-transport-https Gagal Install

**Gejala:**
```
E:Release file is not valid yet (invalid for another 1d 17h...)
```

**Penyebab:** Jam sistem VM tidak sinkron — tertinggal beberapa hari dari waktu sebenarnya.

**Solusi:**
```bash
sudo apt install chrony -y
sudo chronyc makestep
timedatectl status  # verifikasi "System clock synchronized: yes"
```

---

## Verifikasi Security

### Test 1: Kafka Broker TLS

```bash
openssl s_client -connect cp-node1:9092
```

Output yang diharapkan:
```
SSL handshake has read ... bytes
Verification: OK
```

---

### Test 2: Kafka Authentication

Buat file konfigurasi client:

```bash
cat > ~/client.properties << EOF
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="admin" \
  password="admin-secret";
ssl.truststore.location=/var/ssl/private/kafka_broker.truststore.jks
ssl.truststore.password=confluenttruststorepass
EOF
```

**Test Produce:**

```bash
kafka-console-producer \
  --bootstrap-server cp-node1:9092 \
  --topic secure-topic \
  --producer.config ~/client.properties
```

**Test Consume:**

```bash
kafka-console-consumer \
  --bootstrap-server cp-node1:9092 \
  --topic secure-topic \
  --from-beginning \
  --consumer.config ~/client.properties
```

---

### Test 3: ACL Authorization

Tambahkan ACL untuk user `connect`:

```bash
kafka-acls \
  --bootstrap-server cp-node1:9092 \
  --command-config ~/client.properties \
  --add \
  --allow-principal User:connect \
  --operation Read \
  --operation Write \
  --topic connect-cluster-configs
```

Verifikasi ACL:

```bash
kafka-acls \
  --bootstrap-server cp-node1:9092 \
  --command-config ~/client.properties \
  --list \
  --topic connect-cluster-configs
```

---

### Test 4: Unauthorized Access

Coba akses tanpa credential — harus ditolak:

```bash
kafka-console-consumer \
  --bootstrap-server cp-node1:9092 \
  --topic secure-topic
```

Output yang diharapkan:
```
ERROR Error processing message, terminating consumer process:
org.apache.kafka.common.errors.SaslAuthenticationException: SASL authentication failed
```

---

### Test 5: Control Center

Akses via browser:

```
https://192.168.56.10:9021
```

<img width="1897" height="959" alt="image" src="https://github.com/user-attachments/assets/da66d833-ba8a-4747-bf39-8aeea1c95832" />

> **Catatan:** Browser akan menampilkan warning "Not Secure" atau "Your connection is not private" karena menggunakan self-signed certificate. Ini normal — klik **Advanced → Proceed to 192.168.56.10**.

Verifikasi tampilan Control Center:

- **1 Healthy clusters** — cluster Kafka terdeteksi
- **0 Unhealthy clusters** — tidak ada masalah
- **Brokers: 3** — ketiga broker aktif
- **ksqlDB clusters: 1** — ksqlDB terhubung
- **Connect clusters: 1** — Kafka Connect terhubung

---

## Checklist Verifikasi

```
SECURITY
[x] TLS aktif di semua komponen
[x] SASL authentication berhasil
[x] Kafka ACL berfungsi
[x] Unauthorized client ditolak

SERVICES
[x] ZooKeeper TLS aktif
[x] Kafka Broker SASL_SSL aktif
[x] Schema Registry TLS aktif
[x] Kafka Connect TLS aktif
[x] ksqlDB TLS aktif
[x] REST Proxy TLS aktif
[x] Control Center TLS aktif

SECRETS
[x] Semua password tersimpan di Ansible Vault
```

---

## Hasil Akhir

Cluster sekarang memiliki full security stack:

```
Encryption     : TLS (auto-generated cert per node)
Authentication : SASL/PLAIN
Authorization  : Kafka ACL + super.users untuk service accounts
Secrets        : Ansible Vault (enkripsi AES256)
Monitoring     : Control Center via https://192.168.56.10:9021
```

### Ringkasan Konfigurasi Final

| File | Perubahan Kunci |
|---|---|
| `vars/security.yml` | `ssl_enabled: true`, `sasl_protocol: plain`, `super.users` |
| `vars/platform.yml` | Hapus `ssl_enabled: false`, semua URL schema registry ke `https://` |
| `inventory/hosts.yml` | Tambah semua `sasl_users` dengan password dari vault |
| `vault/secrets.yml` | Password semua service account, terenkripsi AES256 |
