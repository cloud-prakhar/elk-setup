# ELK Stack — Docker Setup on WSL2

This repository contains step-by-step guides for setting up and managing the ELK Stack (Elasticsearch + Kibana) on Docker running inside WSL2 on Windows.

---

## Documents

### 1. [ELK Single Node Setup on Docker + WSL2](https://github.com/cloud-prakhar/elk-setup/blob/main/elk-single-node-docker-wsl2-setup.md)

**File:** `elk-single-node-docker-wsl2-setup.md`

A beginner-friendly, step-by-step guide to install and run a single-node ELK Stack using Docker containers on WSL2. Covers everything from pulling images to logging into Kibana as your single unified view over Elasticsearch.

**What's inside:**
- What ELK Stack is and what each component does
- Prerequisites and Docker verification
- Step 1: Pull Docker images
- Step 2: Create Docker network
- Step 3: Start Elasticsearch
- Step 4: Copy the SSL certificate
- Step 5: Reset passwords (`elastic` and `kibana_system`)
- Step 6: Start Kibana (connected to Elasticsearch)
- Step 7: Verify and open Kibana — your single view
- Complete quick-start shell script
- How to stop and restart the stack
- 8 troubleshooting scenarios with fixes (WSL2-specific issues included)
- Useful commands cheat sheet

---

### 2. [ELK Stack Complete Teardown and Uninstall](https://github.com/cloud-prakhar/elk-setup/blob/main/elk-stack-complete-teardown.md)

**File:** `elk-stack-complete-teardown.md`

A complete guide to stop, remove, and clean up every trace of ELK Stack from your machine. Organised into 6 levels — from simply pausing the services all the way to removing Docker itself.

**What's inside:**
- How to check what's running before you delete anything
- Level 1: Stop services (reversible — no data lost)
- Level 2: Remove containers
- Level 3: Remove the Docker network
- Level 4: Remove Docker images (free up ~1.5 GB disk space)
- Level 5: Remove local certificates and files
- Level 6: Remove Docker volumes
- All-in-one complete uninstall script with verification
- Optional: Remove Docker itself from WSL2
- Quick reference table (what each command removes, whether data is lost)

---

## Stack Details

| Component | Container | Port | URL |
|---|---|---|---|
| Elasticsearch | `es01` | `9200` | `https://localhost:9200` |
| Kibana | `kib01` | `5601` | `http://localhost:5601` |

**Login:** `elastic` / *(password set during setup — see Step 5 of the setup guide)*
