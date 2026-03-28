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
- 9 troubleshooting scenarios with fixes (WSL2-specific issues included)
  - Includes: Stack Monitoring "missing encryption key" error fix
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

### 3. [Integrating a Docker App with ELK — Log Shipping & Kibana Views](https://github.com/cloud-prakhar/elk-setup/blob/main/elk-app-log-integration.md)

**File:** `elk-app-log-integration.md`

A practical end-to-end guide showing how to connect a real Docker app (Flask) to the running ELK stack so its logs flow into Elasticsearch and become searchable in Kibana. Includes Kibana Data Views, search queries, and dashboard creation.

**What's inside:**
- How the full pipeline works: App → Filebeat → Logstash → Elasticsearch → Kibana (with diagram)
- Step 1: Copy the CA cert into your app project
- Step 2: `.env` file for credentials (and why not to commit it)
- Step 3: `docker-compose.yml` explained line-by-line (no embedded ES/Kibana — uses external stack)
- Step 4: `filebeat.yml` explained — Docker autodiscovery via container labels
- Step 5: `logstash.conf` explained — JSON parsing, field renaming, daily index output
- Step 6: Start the app stack and verify all 5 containers are running
- Step 7: Generate logs by hitting app endpoints
- Step 8: Verify logs reached Elasticsearch (`_cat/indices`)
- Step 9: Create a Kibana Data View and search in Discover
- KQL and Lucene search query examples
- Step 10: Build a simple dashboard (log count over time + log level pie chart)
- Troubleshooting: 401 errors, missing logs, network issues, Kibana showing no data

---

### 4. [Kibana Sample Queries — KQL, Dev Tools, and Saved Searches](https://github.com/cloud-prakhar/elk-setup/blob/main/elk-kibana-sample-queries.md)

**File:** `elk-kibana-sample-queries.md`

A ready-to-use query reference for exploring logs in Kibana and Elasticsearch. Generic queries that work across any application sending structured JSON logs to this ELK stack — use with demo-app, notes-app, or your own apps.

**What's inside:**
- KQL queries: filter by log level, endpoint, status code, duration, request_id
- Boolean combinations: AND, OR, NOT, range filters
- Dev Tools (Elasticsearch DSL): count, search, aggregations, date histograms
- Aggregation recipes: log level distribution, average response time per endpoint, error rate over time
- Request tracing queries using `request_id`
- Cluster and index health checks
- Saved search suggestions for Kibana Discover
- Dashboard filter reference
- Troubleshooting queries (empty index, wrong timestamp, missing fields)
- Quick reference card

---

## Stack Details

| Component | Container | Port | URL |
|---|---|---|---|
| Elasticsearch | `es01` | `9200` | `https://localhost:9200` |
| Kibana | `kib01` | `5601` | `http://localhost:5601` |

**Login:** `elastic` / *(password set during setup — see Step 5 of the setup guide)*
