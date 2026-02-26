# Task 1: Install Ansible on Ansible Control Node
## Automate Confluent Deployment with Ansible

---

## Topologi Lab (VirtualBox)

```
┌─────────────────────────────────────────────────────────────────┐
│                        VirtualBox Host                          │
│                                                                 │
│  ┌──────────────────┐    ┌──────────────────────────────────┐  │
│  │  Ansible Control │    │         Cluster Nodes            │  │
│  │      Node        │    │                                  │  │
│  │                  │    │  ┌──────────┐  ┌──────────┐      │  │
│  │  cp-ansible      │SSH │  │ cp-node1 │  │ cp-node2 │      │  │
│  │  192.168.56.10   │───►│  │192.168.  │  │192.168.  │      │  │
│  │                  │    │  │56.11     │  │56.12     │      │  │
│  │  - Ansible 9.x   │    │  └──────────┘  └──────────┘      │  │
│  │  - Python 3.10   │    │                                  │  │
│  │  - cp-ansible    │    │  ┌──────────┐                    │  │
│  └──────────────────┘    │  │ cp-node3 │                    │  │
│                          │  │192.168.  │                    │  │
│                          │  │56.13     │                    │  │
│                          │  └──────────┘                    │  │
│                          └──────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Spesifikasi Node

| Node | Hostname | IP Address | Role | User |
|------|----------|------------|------|------|
| Ansible Control | cp-ansible | 192.168.56.10 | Ansible Controller | cpadmin |
| Node 1 | cp-node1 | 192.168.56.11 | Kafka Broker + ZooKeeper | cpadmin |
| Node 2 | cp-node2 | 192.168.56.12 | Kafka Broker + ZooKeeper | cpadmin |
| Node 3 | cp-node3 | 192.168.56.13 | Kafka Broker + ZooKeeper | cpadmin |

> **Catatan:** Sesuaikan IP address dengan konfigurasi VirtualBox Host-Only Network Anda.

---

## Versi yang Digunakan

Berdasarkan dokumentasi resmi Confluent Ansible 7.9:

| Komponen | Versi |
|----------|-------|
| Confluent Platform | 7.9.x |
| Confluent Ansible | 7.9.x |
| Ansible | 9.x (ansible-core 2.16) |
| Python | 3.10.x |
| OS (semua node) | Ubuntu 22.04 LTS |

---

## Persiapan VirtualBox

### Step 1: Konfigurasi Network VirtualBox

Pastikan semua VM memiliki dua network adapter:

1. **Adapter 1:** NAT (untuk akses internet — install package)
2. **Adapter 2:** Host-Only Adapter (untuk komunikasi antar node)

**Buat Host-Only Network di VirtualBox:**
```
VirtualBox Menu → File → Host Network Manager → Create
  IP Address  : 192.168.56.1
  Subnet Mask : 255.255.255.0
  DHCP Server : Disabled (agar IP statis)
```

### Step 2: Fix DNS — Wajib di Semua Node

> ⚠️ Jika VM pernah dibuat menggunakan jaringan kantor, DNS internal kantor ikut tersimpan dan menyebabkan gagal resolve domain saat berganti ke jaringan lain (hotspot, rumah, dsb).

Lakukan di **semua 4 node**:

```bash
# Edit konfigurasi systemd-resolved
sudo nano /etc/systemd/resolved.conf
```

Ubah bagian `[Resolve]` menjadi:
```ini
[Resolve]
DNS=8.8.8.8 8.8.4.4
FallbackDNS=1.1.1.1
DNSStubListener=yes
```

```bash
# Terapkan
sudo rm /etc/resolv.conf
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
sudo systemctl restart systemd-resolved

# Verifikasi
ping -c 3 google.com
```

> **Aman di semua jaringan:** Saat pakai WiFi kantor, DNS kantor dari DHCP tetap jadi prioritas utama. Saat pakai hotspot/jaringan lain, otomatis fallback ke 8.8.8.8.

---

## Konfigurasi Semua Node (Lakukan di SEMUA 4 VM)

### Step 3: Set Hostname

**Di cp-ansible:**
```bash
sudo hostnamectl set-hostname cp-ansible
```

**Di cp-node1:**
```bash
sudo hostnamectl set-hostname cp-node1
```

**Di cp-node2:**
```bash
sudo hostnamectl set-hostname cp-node2
```

**Di cp-node3:**
```bash
sudo hostnamectl set-hostname cp-node3
```

### Step 4: Konfigurasi IP Statis (Semua Node)

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

**Contoh untuk `cp-ansible` (192.168.56.10):**
```yaml
network:
  version: 2
  ethernets:
    enp0s3:          # Adapter 1 - NAT
      dhcp4: true
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
        search: []
    enp0s8:          # Adapter 2 - Host-Only
      dhcp4: no
      addresses:
        - 192.168.56.10/24
```

> **Sesuaikan IP untuk setiap node:**
> - cp-node1: `192.168.56.11/24`
> - cp-node2: `192.168.56.12/24`
> - cp-node3: `192.168.56.13/24`

```bash
sudo netplan apply

# Verifikasi
ip addr show enp0s8
```

### Step 5: Konfigurasi /etc/hosts (Semua Node)

```bash
sudo nano /etc/hosts
```

Tambahkan baris berikut:
```
192.168.56.10   cp-ansible
192.168.56.11   cp-node1
192.168.56.12   cp-node2
192.168.56.13   cp-node3
```

### Step 6: Update Sistem (Semua Node)

```bash
sudo apt update && sudo apt upgrade -y
```

### Step 7: Set Locale — Wajib Persyaratan Confluent (Semua Node)

Confluent Ansible **mensyaratkan** locale `en_US.UTF-8`:

```bash
sudo locale-gen en_US.UTF-8
sudo update-locale LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8
```

Verifikasi (perlu logout/login ulang agar aktif):
```bash
logout
# login kembali, lalu:
locale | grep LANG
# Output: LANG=en_US.UTF-8 ✅
```

### Step 8: Verifikasi Sinkronisasi Waktu — Wajib Persyaratan Confluent (Semua Node)

Ubuntu 22.04 sudah menyertakan `systemd-timesyncd` secara default sehingga **tidak perlu install chrony**. Cukup verifikasi statusnya:

```bash
timedatectl status
```

Output yang diharapkan:
```
               Local time: Wed 2026-02-25 16:03:57 WIB
           Universal time: Wed 2026-02-25 09:03:57 UTC
                 RTC time: Wed 2026-02-25 09:03:39
                Time zone: Asia/Jakarta (WIB, +0700)
System clock synchronized: yes          ← harus YES ✅
              NTP service: active        ← harus active ✅
          RTC in local TZ: no
```

Jika `NTP service: inactive`, aktifkan dengan:
```bash
sudo systemctl enable systemd-timesyncd
sudo systemctl start systemd-timesyncd
sudo timedatectl set-ntp true

# Verifikasi ulang
timedatectl status
```

---

## Instalasi Ansible — Hanya di cp-ansible

> Seluruh langkah berikut **hanya dilakukan di `cp-ansible`**

### Step 9: Verifikasi Python 3.10

Ubuntu 22.04 sudah menyertakan Python 3.10 secara default:

```bash
python3 --version
# Output: Python 3.10.x ✅
```

### Step 10: Install pip

```bash
sudo apt install -y python3-pip

# Upgrade pip ke versi terbaru
python3 -m pip install --upgrade pip
```

### Step 11: Tambahkan PATH

Binary ansible akan terinstall di `~/.local/bin`. Tambahkan ke PATH agar bisa dipanggil langsung:

```bash
echo 'export PATH=$PATH:$HOME/.local/bin' >> ~/.bashrc
source ~/.bashrc
```

### Step 12: Install Ansible 9.x

```bash
# Install Ansible package (bukan ansible-core)
python3 -m pip install "ansible==9.*"
```

> **Mengapa Ansible package, bukan ansible-core?**
> Dokumentasi resmi Confluent merekomendasikan `ansible` package karena sudah menyertakan semua modules dan plugins yang dibutuhkan. `ansible-core` hanya berisi modul minimal dan memerlukan install modul tambahan secara manual.

### Step 13: Verifikasi Instalasi Ansible

```bash
ansible --version
```
<img width="1408" height="242" alt="image" src="https://github.com/user-attachments/assets/5db17405-5259-4be6-91f5-b0d89d341762" />

Output yang diharapkan:
```
ansible [core 2.16.x]
  config file = None
  configured module search path = [...]
  executable location = /home/cpadmin/.local/bin/ansible
  python version = 3.10.x
  jinja version = 3.x.x
  libyaml = True
```

Pastikan:
- `ansible [core 2.16.x]` ✅
- `python version = 3.10.x` ✅

### Step 14: Install Dependencies Tambahan

```bash
pip3 install --user \
  requests \
  jmespath \
  netaddr
```
<img width="1023" height="369" alt="image" src="https://github.com/user-attachments/assets/c32dc862-088b-4cbf-80e6-a7e16fcab78c" />

---

## Konfigurasi SSH Key — cp-ansible ke Semua Node

> Confluent Ansible berkomunikasi via SSH. Control node harus bisa SSH ke semua node **tanpa password**.

### Step 15: Generate SSH Key di cp-ansible

```bash
# Pastikan login sebagai cpadmin
whoami
# Output: cpadmin

ssh-keygen -t rsa -b 4096 -C "cp-ansible" -f ~/.ssh/id_rsa -N ""
```
<img width="1117" height="524" alt="image" src="https://github.com/user-attachments/assets/c974cea1-1b6c-458c-89ba-6a1188a1e1a9" />

Output:
```
Generating public/private rsa key pair.
Your identification has been saved in /home/cpadmin/.ssh/id_rsa
Your public key has been saved in /home/cpadmin/.ssh/id_rsa.pub
```

### Step 16: Copy SSH Key ke Semua Node

```bash
ssh-copy-id cpadmin@cp-node1
ssh-copy-id cpadmin@cp-node2
ssh-copy-id cpadmin@cp-node3
```
<img width="861" height="709" alt="image" src="https://github.com/user-attachments/assets/17cf7254-6cd6-4e93-adf5-71e62fb99119" />

Masukkan password `cpadmin` saat diminta.

### Step 17: Verifikasi SSH Tanpa Password

```bash
ssh cpadmin@cp-node1 "hostname && whoami"
ssh cpadmin@cp-node2 "hostname && whoami"
ssh cpadmin@cp-node3 "hostname && whoami"
```
<img width="785" height="224" alt="image" src="https://github.com/user-attachments/assets/453f1b17-e56a-433c-a719-27aaf576778e" />

Output yang diharapkan:
```
cp-node1
cpadmin
```

### Step 18: Konfigurasi sudo Tanpa Password — Wajib untuk Ansible

Lakukan di **setiap node (cp-node1, cp-node2, cp-node3)**:

```bash
sudo visudo
```

Tambahkan baris berikut di **akhir file**:
```
cpadmin ALL=(ALL) NOPASSWD: ALL
```

Simpan dengan `Ctrl+X → Y → Enter`.

**Verifikasi dari cp-ansible:**
```bash
ssh cpadmin@cp-node1 "sudo whoami"
# Output: root (tanpa diminta password) ✅

ssh cpadmin@cp-node2 "sudo whoami"
# Output: root ✅

ssh cpadmin@cp-node3 "sudo whoami"
# Output: root ✅
```
<img width="1543" height="297" alt="image" src="https://github.com/user-attachments/assets/316226cc-7b9d-46b4-ba63-47722ab4ac1e" />

---

## Install Confluent Ansible Collection

### Step 19: Install via Ansible Galaxy

```bash
ansible-galaxy collection install confluent.platform:==7.9.5
```
<img width="1861" height="319" alt="image" src="https://github.com/user-attachments/assets/38ed5d80-1eb8-4287-ac89-18e31c5a09ff" />

Output yang diharapkan:
```
Starting galaxy collection install process
Process install dependency map
Starting collection install process
confluent.platform (7.9.5) was installed successfully
```

### Step 20: Verifikasi Collection

```bash
ansible-galaxy collection list | grep confluent
```
<img width="809" height="74" alt="image" src="https://github.com/user-attachments/assets/4ae8317d-cb20-4514-8fb5-e5bcb82ec5ce" />

Output:
```
confluent.platform    7.9.5 ✅
```

---

## Buat Struktur Project

### Step 21: Buat Direktori Kerja

```bash
mkdir -p ~/confluent-ansible/{inventory,vars,certs}
cd ~/confluent-ansible
```

### Step 22: Buat File Inventory

```bash
cat > ~/confluent-ansible/inventory/hosts.yml << 'EOF'
---
all:
  vars:
    ansible_user: cpadmin
    ansible_become: true
    ansible_ssh_private_key_file: ~/.ssh/id_rsa

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
EOF
```

---

## Verifikasi Akhir

### Step 23: Test Ansible Ping ke Semua Node

```bash
cd ~/confluent-ansible
ansible -i inventory/hosts.yml all -m ping
```
<img width="953" height="690" alt="image" src="https://github.com/user-attachments/assets/a46d9ce5-ef4f-4c0f-b741-73a94ddc16a5" />

Output yang diharapkan:
```
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

Semua node `SUCCESS` → ✅ **Task 1 Selesai!**

---

## Checklist Verifikasi Lengkap

```
[ ] cp-ansible terhubung ke internet (ping google.com berhasil)
[ ] Semua node bisa saling ping via IP Host-Only (192.168.56.x)
[ ] DNS fix diterapkan di semua node (systemd-resolved + 8.8.8.8)
[ ] /etc/hosts terkonfigurasi di semua node
[ ] Locale en_US.UTF-8 aktif di semua node
[ ] timedatectl: System clock synchronized: yes di semua node
[ ] Python 3.10.x tersedia (python3 --version)
[ ] pip terinstall di cp-ansible
[ ] Ansible 9.x terinstall di cp-ansible (ansible --version)
[ ] PATH sudah include ~/.local/bin
[ ] SSH key: cp-ansible → cp-node1,2,3 tanpa password
[ ] sudo NOPASSWD: cpadmin di cp-node1,2,3
[ ] confluent.platform 7.9.5 terinstall
[ ] ansible ping ke semua node: SUCCESS
```

---

## Troubleshooting

### DNS Gagal / Tidak Bisa Ping Domain
```bash
sudo nano /etc/systemd/resolved.conf
# Pastikan: DNS=8.8.8.8 8.8.4.4

sudo rm /etc/resolv.conf
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
sudo systemctl restart systemd-resolved
ping -c 3 google.com
```

### NTP Tidak Aktif
```bash
sudo systemctl enable systemd-timesyncd
sudo systemctl start systemd-timesyncd
sudo timedatectl set-ntp true
timedatectl status
```

### SSH Permission Denied
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
sudo systemctl restart sshd
```

### Ansible Command Not Found
```bash
echo 'export PATH=$PATH:$HOME/.local/bin' >> ~/.bashrc
source ~/.bashrc
ansible --version
```

### Ansible Collection Not Found
```bash
ansible-galaxy collection install confluent.platform:==7.9.5 --force
ansible-galaxy collection list | grep confluent
```

---

## Ringkasan Versi Terinstall

| Komponen | Perintah Verifikasi | Expected Output |
|----------|---------------------|-----------------|
| Python | `python3 --version` | `Python 3.10.x` |
| Ansible | `ansible --version` | `ansible [core 2.16.x]` |
| Collection | `ansible-galaxy collection list \| grep confluent` | `confluent.platform 7.9.5` |
| NTP | `timedatectl status` | `System clock synchronized: yes` |
| SSH | `ssh cpadmin@cp-node1 hostname` | `cp-node1` |

---


> **Referensi Resmi:** [Confluent Ansible 7.9 Requirements](https://docs.confluent.io/ansible/7.9/ansible-requirements.html#general-requirements)

