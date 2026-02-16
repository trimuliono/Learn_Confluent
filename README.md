# Learn Confluent

### Apache Kafka & Confluent Platform 7.9.x Hands-On Lab Series

This repository documents a structured hands-on learning journey for mastering **Apache Kafka** and **Confluent Platform 7.9.x**, from foundational setup to end-to-end event streaming architecture.

The project is designed to simulate real-world implementation scenarios, focusing on installation, configuration, integration, and stream processing using Confluent ecosystem components.

---

## 📌 Project Overview

This learning series is divided into phases:

| Phase   | Focus Area                                   | Outcome                                          |
| ------- | -------------------------------------------- | ------------------------------------------------ |
| Phase 1 | Platform Installation & Core Services        | Fully operational single-node Confluent Platform |
| Phase 2 | End-to-End Streaming (Avro, Connect, ksqlDB) | Complete streaming data pipeline                 |

Each phase includes hands-on implementation and technical documentation.

---

# 🔹 Phase 1 – Confluent Platform Installation & Exploration

**Objective:**
Deploy and validate a complete Confluent Platform 7.9.x environment.

### Covered Components

* Zookeeper
* Kafka Broker
* Schema Registry
* Kafka Connect
* ksqlDB
* Kafka REST Proxy
* Control Center

### Key Activities

* Installation via package manager
* Service deployment using systemd
* Cluster validation (quorum & cluster ID check)
* Topic creation
* CLI producer & consumer testing
* Full service health verification

📄 Documentation:

```
Installing CP 7.9.x Via package manager.md
```

---

# 🔹 Phase 2 – End-to-End Streaming Pipeline

**Objective:**
Build a real streaming architecture using Avro, Kafka Connect, and ksqlDB.

### Architecture Scope

* Avro-based message serialization
* Schema Registry integration
* Source Connector
* Sink Connector
* Stream processing using ksqlDB
* End-to-end data flow validation

### Implementation Highlights

* Topic creation with Avro schema
* Producer & Consumer with schema validation
* Kafka Connect source & sink configuration
* Stream transformation using ksqlDB
* Data pipeline verification

📄 Documentation:

```
CP 7.9.x-avro-connect-ksqldb-end-to-end-streaming-lab.md
```

---

# 🏗 Architecture Summary

This repository demonstrates an event streaming architecture composed of:

```
Producer → Kafka Topic (Avro) → 
Schema Registry → 
Kafka Connect (Source/Sink) → 
ksqlDB Stream Processing → 
Downstream System
```

The setup reflects real enterprise streaming patterns using Confluent Platform components.

---

# 🛠 Technical Stack

* Confluent Platform 7.9.x
* Apache Kafka
* Apache Zookeeper
* Schema Registry
* Kafka Connect
* ksqlDB
* Avro Serialization
* Linux (systemd-based deployment)

---

# 🎯 Learning Outcomes

After completing this project, the practitioner will be able to:

* Install and configure Confluent Platform from scratch
* Understand Kafka cluster architecture
* Manage Kafka topics and brokers
* Implement Avro schema management
* Build streaming pipelines using Kafka Connect
* Perform stream processing with ksqlDB
* Validate and troubleshoot Kafka services

---

# 📂 Repository Structure

```
Learn_Confluent/
│
├── Introduction.md
├── Lab 01 Exploring Apache Kafka.md
├── Installing CP 7.9.x Via package manager.md
├── CP 7.9.x-avro-connect-ksqldb-end-to-end-streaming-lab.md
└── README.md
```

---

# 🚀 Future Enhancements

Planned improvements for upcoming phases:

* TLS & SASL Security implementation
* ACL configuration
* Multi-broker cluster setup
* Monitoring & observability integration
* Production-grade deployment patterns

---

# 👤 Author

Self-paced hands-on implementation as part of continuous learning in event streaming and distributed systems.

---
