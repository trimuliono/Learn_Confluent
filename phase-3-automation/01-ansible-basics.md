# 01 – Ansible Fundamentals for Confluent Platform Automation

## 🎯 Objective

This section establishes the foundational knowledge required to automate Confluent Platform deployment using Ansible.

The goal is to:

* Understand Ansible architecture
* Prepare a multi-node inventory
* Configure SSH-based orchestration
* Execute basic playbooks
* Validate idempotent automation behavior
* Prepare structure for Phase 3 automation roles

This phase focuses strictly on automation fundamentals before introducing Confluent-specific deployment logic.

---

# 🧠 1. What is Ansible?

Ansible is an agentless IT automation tool used for:

* Configuration management
* Application deployment
* Infrastructure provisioning
* Orchestration
* Security automation

It operates over SSH and does not require agents installed on target nodes.

---

# 🏗 2. Architecture Overview

Ansible uses a simple control model:

```
Control Node
   │
   ├── SSH → Kafka Node 1
   ├── SSH → Kafka Node 2
   ├── SSH → Kafka Node 3
   ├── SSH → Zookeeper Node 1
   └── ...
```

### Key Components

| Component     | Description                             |
| ------------- | --------------------------------------- |
| Control Node  | Machine where Ansible is installed      |
| Managed Nodes | Target servers (Kafka, Zookeeper, etc.) |
| Inventory     | List of target hosts                    |
| Playbook      | YAML automation definition              |
| Module        | Built-in task logic                     |
| Role          | Reusable automation structure           |

---

# 🖥 3. Lab Environment Preparation

## 3.1 Target Architecture (Phase 3 Lab)

For automation testing, we simulate:

* 3 ZooKeeper nodes
* 3 Kafka brokers
* (Optional) Schema Registry, Connect, ksqlDB

Example Host Layout:

| Host  | Role         | IP            |
| ----- | ------------ | ------------- |
| node1 | zk1 + kafka1 | 192.168.56.11 |
| node2 | zk2 + kafka2 | 192.168.56.12 |
| node3 | zk3 + kafka3 | 192.168.56.13 |

---

# 📦 4. Install Ansible on Control Node

## Ubuntu / Debian

```bash
sudo apt update
sudo apt install ansible -y
```

Verify:

```bash
ansible --version
```

Expected:

```
ansible [core X.X.X]
python version = 3.X
```

---

# 🔐 5. Configure SSH Access

Ansible requires passwordless SSH access.

## 5.1 Generate SSH Key

```bash
ssh-keygen -t rsa -b 4096
```

## 5.2 Copy Key to All Nodes

```bash
ssh-copy-id user@192.168.56.11
ssh-copy-id user@192.168.56.12
ssh-copy-id user@192.168.56.13
```

Test connectivity:

```bash
ssh user@192.168.56.11
```

---

# 📁 6. Create Project Structure

Inside your repository:

```bash
mkdir -p infra/ansible
cd infra/ansible
```

Create structure:

```
infra/ansible/
├── inventory/
│   └── hosts.yml
├── playbooks/
│   └── ping-test.yml
├── roles/
├── group_vars/
└── host_vars/
```

---

# 📋 7. Define Inventory

Create `inventory/hosts.yml`:

```yaml
all:
  children:
    zookeeper:
      hosts:
        zk1:
          ansible_host: 192.168.56.11
        zk2:
          ansible_host: 192.168.56.12
        zk3:
          ansible_host: 192.168.56.13

    kafka:
      hosts:
        kafka1:
          ansible_host: 192.168.56.11
        kafka2:
          ansible_host: 192.168.56.12
        kafka3:
          ansible_host: 192.168.56.13
```

Test inventory:

```bash
ansible-inventory -i inventory/hosts.yml --list
```

---

# 🧪 8. Test Connectivity (Ping Module)

Create `playbooks/ping-test.yml`:

```yaml
- name: Test connectivity to all nodes
  hosts: all
  gather_facts: false

  tasks:
    - name: Ping nodes
      ansible.builtin.ping:
```

Run:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/ping-test.yml
```

Expected:

```
ok: [zk1]
ok: [zk2]
ok: [zk3]
ok: [kafka1]
...
```

---

# 🔁 9. Understanding Idempotency

Ansible is idempotent.

Meaning:

* Running playbook multiple times
* Produces same final state
* Without unnecessary changes

Example:

```yaml
- name: Install curl
  hosts: all
  become: true

  tasks:
    - name: Install curl package
      ansible.builtin.apt:
        name: curl
        state: present
```

First run:

```
changed: [node1]
```

Second run:

```
ok: [node1]
```

This behavior is critical for production automation.

---

# 🏗 10. Basic Playbook Structure Explained

```yaml
- name: Example play
  hosts: kafka
  become: true

  vars:
    example_variable: value

  tasks:
    - name: Task description
      module_name:
        parameter: value
```

Key Elements:

| Element  | Function                            |
| -------- | ----------------------------------- |
| hosts    | Target group                        |
| become   | Privilege escalation                |
| vars     | Variables                           |
| tasks    | Execution steps                     |
| handlers | Triggered actions (restart service) |

---

# 🧱 11. Introducing Roles (Preview for Phase 3)

Roles allow modular automation.

Example future structure:

```
roles/
├── zookeeper/
│   ├── tasks/
│   ├── templates/
│   └── handlers/
└── kafka/
```

Roles improve:

* Reusability
* Maintainability
* Clean separation of concerns

We will implement this in Phase 3 step 02.

---

# ⚠️ Common Beginner Mistakes

| Mistake                        | Explanation                  |
| ------------------------------ | ---------------------------- |
| Using IP directly in playbooks | Should use inventory groups  |
| Hardcoding passwords           | Use Ansible Vault            |
| Ignoring idempotency           | Leads to unstable automation |
| No handler usage               | Causes unnecessary restarts  |

---

# 📌 Phase 3 Readiness Checklist

Before proceeding to deployment automation:

* [ ] SSH connectivity verified
* [ ] Inventory validated
* [ ] Playbook executed successfully
* [ ] Idempotency behavior observed
* [ ] Directory structure prepared

---

# 🚀 Next Step

Proceed to:

```
02-deploy-cp-no-security.md
```

Where we automate Confluent Platform installation without security, mirroring Phase 1 but fully automated.

---

# 📚 Official Documentation References

Ansible Official Documentation
[https://docs.ansible.com/ansible/latest/index.html](https://docs.ansible.com/ansible/latest/index.html)

Ansible Inventory Guide
[https://docs.ansible.com/ansible/latest/inventory_guide/index.html](https://docs.ansible.com/ansible/latest/inventory_guide/index.html)

Ansible Playbook Guide
[https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_intro.html](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_intro.html)

Ansible Best Practices
[https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_best_practices.html](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_best_practices.html)

---
