## The Problem
Di dunia saat ini, bisnis menghasilkan data real-time dalam jumlah besar. seperti data transaksi pengguna e-commerce, data transaksi bank digital, data user gaming, pembacaan sensor, dan masih banyak lainnya.
metode data prosesing tradisional sering kali kesulitan mengelola data real-time menyebabkan keterlambata informasi, hambatan sistem, dan kesulitan integrasi.

Seiring berkembangnya bisnis, mereka mengumpulkan lebih banyak data dari berbagai sumber dan menyimpannya di berbagai tujuan. Hasilnya seiring berjalannya waktu adalah "spaghetti mess" koneksi dan penyimpanan data yang sulit dikelola.
Memberikan prediksi kinerja, apalagi jaminan, dalam kerumitan seperti ini merupakan suatu tantangan.
<img width="625" height="458" alt="image" src="https://github.com/user-attachments/assets/91a3e01d-38d4-4d45-a7da-42ce4544063c" />

Apache kafka menguraikan "spaghetti mesh" dengan berfungsi sebagai titik pusat distribusi untuk koneksi. Sehingga setiap produser maupun konsumer terhubung independen pada kafka cluster.
Penerapan apache kafka oleh confluent menambahkan manajemen, tata kelola, keamanan, dan fitur tambahan untuk skala bisnis ernterprise.
<img width="616" height="387" alt="image" src="https://github.com/user-attachments/assets/34985d5f-ba78-48f9-ad3b-d1cee95c306b" />

---

### Apa itu apache kafka

Apache kafka adalah **platform event streaming terdistribusi** yang digunakan untuk:

* Mempublikasikan (publish) event
* Berlangganan (subcribe) event
* Menyimpan event secara durable
* Memproses event secara real-time maupun batch

- kafka adalah sebuah open source distribusi event streaming platform untuk membangun real-time data pipeline dan aplikasi streaming.
- Platform streaming event terdistribusi, dirancang untuk data pipeline dengan throughtput tinggi, toleran terhadap kesalahan, dan dapat diskalakan.
- Platform streaming event yang memberikan landasan untuk mengumpulkan, memproses, menyimpan dan mengintegrasikan dalam skala besar
- Memiliki penyimpanan data yang tahan lama. karna disimpan on disk dan memiliki pengaturan retensi.
- Dapat brfungsi sebagai single source of truth, dengan memusatkan data dari semua sumber.

---
### Konsep Event Streaming

* **Event** merepresentasikan sesuatu yang terjadi pada suatu waktu (contoh: order dibuat, transaksi berhasil).
* Event streaming memungkingkan sistem:
  * Mengalirkan data secara terus-menerus
  * Memprose data secara real-time
  * Menyimpan data untuk replay di masa depan
* Kafka menyimpan evenr sebagai **log terurut (append-only log)**

---
### Alasan Menggunakan Kafka

Kafka dikembangkan untuk mengatasi keterbatasan sistem tradisional dalam menangani data real-time.

**Masalah yang diselesaikan Kafka:**

* Integrasi antar sistem yang kompleks
* Ketergantungan kuat antar aplikasi (tight coupling)
* Keterbatasan skalabilitas dan performa

**Keunggulan Kafka:**

* High throughput
* Fault tolerant
* Scalable secara horizontal
* Data bisa di-replay
* Mendukung multiple consumer

---
### Use Case Utama Kafka

* **Messaging system** (alternatif message queue)
* **Activity tracking** (user activity, log event)
* **Metics & Monitoring**
* **Data pipeline**
* **Stream Processing**

kafka sering digunakan sebagai **central data backbone** dalam arsitektur modern.

---
### Arsitektir dasar kafka

**Komponen Utama**

* **Producer**: Aplikasi pengirim event
* **Topic**: Kategori/log tmpat event disimpan
* **Partition**: Pembagi data untuk skalabilitas dan oerdering
* **Broker**: Server kafka
* **Consumer**: Aplikasi pembaca Event
* **Consumer Group**: Mekanisme load balancing antar consumer

**Prinsip penting:**

* Data disimpan dalam urutan (ordered per partition)
* Consumer mengelola offset sendiri
* Kafka tidak menghapus data setelah dibaca

---

### 6. Kafka sebagai Distributed System

* Kafka menyimpan data dengan **replication**
* Jika broker gagal, data tetap tersedia
* Kafka mendukung **high availability**
* Cocok untuk sistem mission-critical

---
