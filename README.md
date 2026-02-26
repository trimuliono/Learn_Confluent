# Learn Confluent

### Enterprise Event Streaming Lab – Confluent Platform 7.9.x

This repository documents a structured, hands-on engineering journey for mastering Apache Kafka and Confluent Platform 7.9.x — from installation, to end-to-end streaming implementation, security hardening, and automation.

The project is organized into three progressive phases that simulate real-world enterprise delivery practices.

---

# 📌 Project Phases

---

## 🔹 Phase 1 – Confluent Platform Installation

**Objective:** Deploy and validate a fully functional Confluent Platform cluster.

### Scope

* Install Confluent Platform 7.9.x via package manager
* Configure and start:

  * Zookeeper
  * Kafka Broker
  * Schema Registry
  * Kafka Connect
  * ksqlDB
  * Kafka REST Proxy
  * Control Center
* Deploy services using systemd
* Validate quorum & cluster ID
* Topic creation
* CLI producer & consumer testing

📂 Location:

```
phase-1-installation/
└── Installing CP 7.9.x Via package manager.md
```

---

## 🔹 Phase 2 – End-to-End Streaming & Security

**Objective:** Build a complete streaming pipeline, validate it, then apply enterprise-grade security controls.

---

### 🧩 Part 1 – End-to-End Streaming (Baseline Implementation)

This stage establishes a fully working data pipeline in a non-secure environment before introducing encryption and authentication.

#### End-to-End Streaming Lab

* Avro serialization
* Schema Registry integration
* Kafka Connect (source & sink)
* ksqlDB stream processing
* Complete streaming pipeline validation

📂 Location:

```
phase-2-security/
└── CP 7.9.x-avro-connect-ksqldb-end-to-end-streaming-lab.md
```

This ensures the architecture and data flow are validated before security layers are introduced.

---

### 🔐 Part 2 – Security Hardening

After validating the functional pipeline, security controls are applied across all components.

#### Security Implementation

* TLS encryption
* SASL authentication
* Zookeeper security
* Inter-broker security
* Client security
* ACL configuration
* Secure produce & consume validation

📂 Location:

```
phase-2-security/
└── Kafka Security End-to-End.md
```

---

## 🔹 Phase 3 – Automation & Operational Engineering

**Objective:** Automate deployment and simulate production-grade operational practices.

### Scope

* Ansible fundamentals
* Automated deployment (non-secure)
* Automated secure deployment
* Secret management using Ansible Vault
* Monitoring integration
* Troubleshooting scenarios

📂 Location:

```
phase-3-automation/
├── 01-install-ansible-on-ansible-control-node.md
├── 02-deploy-cp-no-security.md
├── 03-deploy-cp-secure.md
├── 04-ansible-vault.md
├── 05-monitoring.md
└── 06-troubleshooting-scenarios.md
```

---

# 🏗 Architecture Overview

The project demonstrates a complete Confluent-based streaming architecture:

```
Producer
   ↓
Kafka Broker (Avro)
   ↓
Schema Registry
   ↓
Kafka Connect (Source / Sink)
   ↓
ksqlDB Stream Processing
   ↓
Downstream Systems
```

Security and automation layers are progressively applied after validating the functional baseline.

---

# 🛠 Technology Stack

* Confluent Platform 7.9.x
* Apache Kafka
* Apache Zookeeper
* Schema Registry
* Kafka Connect
* ksqlDB
* Avro
* TLS / SASL
* ACL
* Ansible
* Linux (systemd-based services)

---

# 🎯 Engineering Outcomes

After completing all three phases, this project demonstrates the ability to:

* Deploy and manage Confluent Platform
* Build and validate an end-to-end streaming pipeline
* Implement Kafka security (TLS, SASL, ACL)
* Automate infrastructure using Ansible
* Apply monitoring and troubleshooting practices
* Structure technical documentation in a production-ready format

---
