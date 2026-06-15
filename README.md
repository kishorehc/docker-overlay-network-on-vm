# Docker Swarm Overlay Network — Project README

## Overview

This project demonstrates deploying a multi-node **Docker Swarm** cluster on AWS EC2 with a custom **overlay network** (`kish-lay`) to enable container-to-container communication across multiple hosts. A replicated Nginx service (`myapp_web`) is deployed across the swarm with 3 replicas distributed among the manager and worker nodes.

---

## Architecture

```
AWS EC2 (ap-south-1 / Mumbai)
│
├── manager   (t3.micro) — Swarm Manager Node
│   └── myapp_web.1  [nginx:latest]
│
├── worker-1  (t3.micro) — Swarm Worker Node
│   └── myapp_web.2  [nginx:latest]
│
└── worker-2  (t3.micro) — Swarm Worker Node
    └── myapp_web.3  [nginx:latest]
```

All nodes are in **Availability Zone:** `ap-south-1a`

---

## Infrastructure

| Node      | Instance ID           | Public IP       | Private IP      |
|-----------|-----------------------|-----------------|-----------------|
| manager   | i-01400b01846ac95ac   | 13.127.178.36   | 172.31.35.118   |
| worker-1  | i-08222d4ec67c5730b   | 13.233.164.153  | 172.31.47.221   |
| worker-2  | i-05d712e4dca79d70c   | 13.233.86.83    | 172.31.37.24    |

All instances: **t3.micro**, running **Ubuntu**, managed via **AWS EC2 Instance Connect**.
<img width="3834" height="2157" alt="Screenshot 2026-06-15 230313" src="https://github.com/user-attachments/assets/a72c2239-f3f7-4f84-a0d7-c26f736cce6f" />
<img width="3837" height="2157" alt="Screenshot 2026-06-15 230321" src="https://github.com/user-attachments/assets/0b9d48d0-208b-464b-b6f5-b42686b2ffa9" />
<img width="3837" height="2157" alt="Screenshot 2026-06-15 230338" src="https://github.com/user-attachments/assets/6b1abe59-ae38-4e94-9a94-83cb2e371abe" />
<img width="3837" height="2157" alt="Screenshot 2026-06-15 230346" src="https://github.com/user-attachments/assets/8204251a-5773-4871-8583-5b69304b83ad" />

---
<img width="3837" height="2148" alt="Screenshot 2026-06-15 230442" src="https://github.com/user-attachments/assets/d2cd7255-f065-4d4c-ad6a-369e412194f1" />

## Networking

### Networks Present on All Nodes

| Network ID     | Name            | Driver  | Scope  |
|----------------|-----------------|---------|--------|
| (varies)       | bridge          | bridge  | local  |
| (varies)       | docker_gwbridge | bridge  | local  |
| (varies)       | host            | host    | local  |
| ksk9falde0tu   | ingress         | overlay | swarm  |
| zq68gzf4dx6x   | kish-lay        | overlay | swarm  |
| (varies)       | none            | null    | local  |

### Key Network: `kish-lay`
- **Driver:** overlay
- **Scope:** swarm
- **Purpose:** Custom overlay network for cross-host container communication
- **Note:** `worker-2` joined the `kish-lay` network after service deployment (visible in final `docker network ls` output)

---

## Service Deployment

### Service: `myapp_web`

| Property   | Value            |
|------------|------------------|
| Image      | nginx:latest     |
| Mode       | Replicated       |
| Replicas   | 3/3 (all running)|
| Port       | 8080 → 80/tcp    |
| Network    | kish-lay         |

### Replica Placement

| Task ID      | Task Name      | Node     | Current State              |
|--------------|----------------|----------|----------------------------|
| 29qxz0xd0sm8 | myapp_web.1    | manager  | Running (46 seconds ago)   |
| hsnlqh61ux61 | myapp_web.2    | worker-1 | Running (46 seconds ago)   |
| np8yg2rp6d20 | myapp_web.3    | worker-2 | Running (39 seconds ago)   |

---

## Docker Compose File (`docker-compose.yml`)

```yaml
version: '3.8'

services:
  web:
    image: nginx:latest
    deploy:
      replicas: 3
      restart_policy:
        condition: any
    networks:
      - kish-lay  # Using your existing overlay network
    ports:
      - "8080:80"

networks:
  kish-lay:
    external: true   # IMPORTANT: tells Docker to use existing network
    name: kish-lay   # Your existing network name
```

> **Note:** `external: true` instructs Docker Swarm to attach the service to the pre-existing `kish-lay` overlay network rather than creating a new one.

---

## Key Concepts

### What is a Docker Overlay Network?
An overlay network spans multiple Docker hosts (nodes), allowing containers on different physical/virtual machines to communicate as if they were on the same local network. This is the backbone of Docker Swarm networking.

### Why `external: true`?
When a network already exists in the swarm (created separately with `docker network create --driver overlay kish-lay`), using `external: true` in the Compose file tells the stack deployment to reuse it instead of creating a new one. This is useful when multiple services or stacks need to share the same network.

### Swarm Manager vs Worker
- **Manager nodes** handle cluster state, scheduling, and accept `docker service` commands.
- **Worker nodes** only run tasks (containers). Running `docker service ls` on a worker returns an error — this is expected behavior.

---

## Commands Used

```bash
# On Manager — List all services
docker service ls

# On Manager — Check replica placement for the service
docker service ps myapp_web

# On any node — List all Docker networks
docker network ls

# Deploy stack using compose file (run on manager)
docker stack deploy -c docker-compose.yml myapp

# Create overlay network manually (run on manager)
docker network create --driver overlay --attachable kish-lay
```

---

## Access

Once deployed, Nginx is accessible on **port 8080** of any node's public IP:

```
http://13.127.178.36:8080   # via manager
http://13.233.164.153:8080  # via worker-1
http://13.233.86.83:8080    # via worker-2
```

> Ensure the EC2 Security Group allows **inbound TCP on port 8080** from your IP or `0.0.0.0/0`.

---

## Author

**Kishore H C** | AWS DevOps Engineer Trainee  
GitHub: [kishorehc](https://github.com/kishorehc) | LinkedIn: [kishore-h-c-847775287](https://linkedin.com/in/kishore-h-c-847775287)
