# ELK Stack Setup on Docker (WSL2 / Single Node)

> **Who is this for?** Anyone — even a complete beginner! If you can copy-paste commands, you can run ELK Stack on your laptop.

---

## What is ELK Stack?

Think of ELK like this:

| Tool | What it does | Simple analogy |
|---|---|---|
| **E**lasticsearch | Stores and searches your data | A super-fast filing cabinet |
| **L**ogstash | Collects and processes data (optional) | A post sorter that organises your mail |
| **K**ibana | Visual dashboard to explore data | A window to look inside the filing cabinet |

---

## What You Need Before Starting

- **Windows** with **WSL2** installed (Ubuntu recommended)
- **Docker Desktop** installed and running on Windows
- **Internet connection** to download images

### Check Docker is working

```bash
docker --version
```

You should see something like `Docker version 27.x.x`. If you get an error, open Docker Desktop on Windows first, then try again.

---

## Overview: What We Will Do

```
Step 1: Pull the Docker images (download Elasticsearch and Kibana)
Step 2: Create a Docker network (so containers can talk to each other)
Step 3: Start Elasticsearch
Step 4: Copy the security certificate
Step 5: Reset the passwords
Step 6: Start Kibana
Step 7: Open Kibana in your browser — single view for everything
```

---

## Step 1: Pull the Docker Images

Before running anything, download the images from the Elastic registry. Think of this like downloading an app installer — you do it once, and it stays on your machine.

```bash
docker pull docker.elastic.co/elasticsearch/elasticsearch:9.3.2
docker pull docker.elastic.co/kibana/kibana:9.3.2
```

**What this does:**
- `docker pull` = download an image from the internet to your local machine
- `docker.elastic.co/...` = the official Elastic image registry (like an app store just for Elastic products)
- `elasticsearch:9.3.2` / `kibana:9.3.2` = the exact version we want (always pin a version so your setup is reproducible)

**Expected output:**
```
9.3.2: Pulling from elasticsearch/elasticsearch
...
Status: Downloaded newer image for docker.elastic.co/elasticsearch/elasticsearch:9.3.2
```

This may take a few minutes depending on your internet speed. Elasticsearch image is about 1 GB.

**Verify the images are downloaded:**
```bash
docker images | grep elastic
```

You should see both `elasticsearch` and `kibana` listed.

---

## Step 2: Create a Docker Network

```bash
docker network create elastic
```

**What this does:**
Docker containers by default cannot talk to each other. This command creates a private network called `elastic` — like a private WiFi network just for our ELK containers. Both Elasticsearch and Kibana will join this network so they can find each other by name.

**Expected output:**
```
a5bdceb71594c...  (a long random ID — this is normal)
```

---

## Step 3: Start Elasticsearch

```bash
docker run --name es01 \
  --net elastic \
  -p 9200:9200 \
  -m 1GB \
  -d \
  docker.elastic.co/elasticsearch/elasticsearch:9.3.2
```

**Breaking down each part:**

| Flag | Meaning |
|---|---|
| `--name es01` | We are naming this container "es01" (Elasticsearch node 1) |
| `--net elastic` | Connect it to our elastic network from Step 2 |
| `-p 9200:9200` | Open port 9200 — this is the door we knock on to talk to Elasticsearch |
| `-m 1GB` | Limit memory to 1 GB so it does not eat your whole computer |
| `-d` | Run in background (detached mode) — so your terminal stays free |

**Wait about 30 seconds** for it to start, then check:

```bash
docker ps
```

You should see `es01` with status `Up X seconds`.

---

## Step 4: Copy the Security Certificate

Elasticsearch uses HTTPS (encrypted connections). It creates its own security certificate inside the container. We need to copy it out so Kibana can use it.

```bash
docker cp es01:/usr/share/elasticsearch/config/certs/http_ca.crt ~/ELK/http_ca.crt
```

**What this does:**
- `docker cp` = copies a file from inside a container to your computer
- `es01:...` = from the container named "es01", at this path inside it
- `~/ELK/http_ca.crt` = save it here on your computer

**Create the folder first if it doesn't exist:**

```bash
mkdir -p ~/ELK
```

---

## Step 5: Reset Passwords

### 5a. Reset the `elastic` superuser password

```bash
echo "y" | docker exec -i es01 \
  /usr/share/elasticsearch/bin/elasticsearch-reset-password \
  -u elastic -s
```

**What this does:**
- `docker exec -i es01` = run a command inside the running container
- `elasticsearch-reset-password -u elastic` = reset the password for the built-in admin user called `elastic`
- `-s` = "silent" — only print the password, nothing else

**Save the password it prints!** It looks like: `WzHN65Wf6V9P*p7rt6Cd`

### 5b. Reset the `kibana_system` password

```bash
echo "y" | docker exec -i es01 \
  /usr/share/elasticsearch/bin/elasticsearch-reset-password \
  -u kibana_system -s
```

This is the internal account Kibana uses to connect to Elasticsearch. Save this password too.

---

## Step 6: Start Kibana

```bash
docker run -d \
  --name kib01 \
  --net elastic \
  -p 5601:5601 \
  -v ~/ELK/http_ca.crt:/usr/share/kibana/config/certs/http_ca.crt \
  -e ELASTICSEARCH_HOSTS=https://es01:9200 \
  -e ELASTICSEARCH_USERNAME=kibana_system \
  -e ELASTICSEARCH_PASSWORD="<your-kibana-system-password>" \
  -e ELASTICSEARCH_SSL_CERTIFICATEAUTHORITIES=/usr/share/kibana/config/certs/http_ca.crt \
  -e ELASTICSEARCH_SSL_VERIFICATIONMODE=none \
  docker.elastic.co/kibana/kibana:9.3.2
```

> Replace `<your-kibana-system-password>` with the password from Step 5b.

**Breaking down each part:**

| Flag | Meaning |
|---|---|
| `-v ~/ELK/http_ca.crt:...` | Mount (inject) the certificate file INTO the container |
| `-e ELASTICSEARCH_HOSTS=...` | Tell Kibana where to find Elasticsearch (by container name `es01`) |
| `-e ELASTICSEARCH_USERNAME=kibana_system` | The account Kibana uses to connect |
| `-e ELASTICSEARCH_PASSWORD=...` | The password for that account |
| `-e ELASTICSEARCH_SSL_CERTIFICATEAUTHORITIES=...` | Path to the CA cert inside the container |
| `-e ELASTICSEARCH_SSL_VERIFICATIONMODE=none` | Skip hostname verification (needed because the cert is issued to the container ID, not `es01`) |

**Wait about 30–60 seconds** for Kibana to start fully.

---

## Step 7: Verify and Open Kibana — Your Single View

### Check containers are up

```bash
docker ps
```

You should see two containers with status `Up`:
```
CONTAINER ID   IMAGE               STATUS          NAMES
xxxxxxxxxx     kibana:9.3.2        Up X seconds    kib01
xxxxxxxxxx     elasticsearch:9.3.2 Up X minutes    es01
```

### Test Elasticsearch is responding

```bash
curl --cacert ~/ELK/http_ca.crt \
  -u elastic:"<your-elastic-password>" \
  https://localhost:9200
```

You should get a JSON response with `"cluster_name": "docker-cluster"`.

### Open Kibana in your browser — Single View for Everything

Kibana is your **single window** into the entire ELK stack. It connects to Elasticsearch in the background and shows you everything in one place: search your data, create dashboards, monitor cluster health, manage indices, and more.

Open your Windows browser and go to:

```
http://localhost:5601
```

Login with:
- **Username:** `elastic`
- **Password:** the password you saved from Step 5a

Once logged in:
- Go to **Management → Stack Monitoring** to see your Elasticsearch cluster health
- Go to **Discover** to search through your data
- Go to **Management → Index Management** to see all indices
- Go to **Dev Tools → Console** to run Elasticsearch queries directly from Kibana's UI

> Kibana is connected to Elasticsearch via the `ELASTICSEARCH_HOSTS` env var we set in Step 6. No extra setup needed — it is already wired up and ready to use.

---

## Complete Quick-Start Script

If you want to run everything at once (after reading the steps above):

```bash
#!/bin/bash
# ELK Stack Quick Start for WSL2

# Create the folder for certs
mkdir -p ~/ELK

# Step 1: Pull images
docker pull docker.elastic.co/elasticsearch/elasticsearch:9.3.2
docker pull docker.elastic.co/kibana/kibana:9.3.2

# Step 2: Create network
docker network create elastic

# Step 3: Start Elasticsearch
docker run --name es01 --net elastic -p 9200:9200 -m 1GB -d \
  docker.elastic.co/elasticsearch/elasticsearch:9.3.2

echo "Waiting 30 seconds for Elasticsearch to start..."
sleep 30

# Step 4: Copy certificate
docker cp es01:/usr/share/elasticsearch/config/certs/http_ca.crt ~/ELK/http_ca.crt

# Step 5: Reset passwords
echo "==> Resetting elastic password..."
ELASTIC_PASS=$(echo "y" | docker exec -i es01 /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic -s)
echo "elastic password: $ELASTIC_PASS"

echo "==> Resetting kibana_system password..."
KIBANA_PASS=$(echo "y" | docker exec -i es01 /usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system -s)
echo "kibana_system password: $KIBANA_PASS"

# Step 6: Start Kibana
docker run -d \
  --name kib01 \
  --net elastic \
  -p 5601:5601 \
  -v ~/ELK/http_ca.crt:/usr/share/kibana/config/certs/http_ca.crt \
  -e ELASTICSEARCH_HOSTS=https://es01:9200 \
  -e ELASTICSEARCH_USERNAME=kibana_system \
  -e ELASTICSEARCH_PASSWORD="$KIBANA_PASS" \
  -e ELASTICSEARCH_SSL_CERTIFICATEAUTHORITIES=/usr/share/kibana/config/certs/http_ca.crt \
  -e ELASTICSEARCH_SSL_VERIFICATIONMODE=none \
  docker.elastic.co/kibana/kibana:9.3.2

echo ""
echo "==================================="
echo " ELK Stack is starting!"
echo "==================================="
echo " Kibana:        http://localhost:5601"
echo " Elasticsearch: https://localhost:9200"
echo " Username:      elastic"
echo " Password:      $ELASTIC_PASS"
echo "==================================="
echo "Wait ~60 seconds for Kibana to fully start."
```

Save this as `start-elk.sh`, then run:
```bash
chmod +x start-elk.sh
./start-elk.sh
```

---

## How to Stop ELK Stack

```bash
docker stop kib01 es01
```

**What this does:** Gracefully stops both containers. Your data is preserved inside the containers.

## How to Start It Again

```bash
docker start es01
sleep 20
docker start kib01
```

> Always start Elasticsearch first and wait for it before starting Kibana.

## How to Completely Remove Everything

```bash
docker stop kib01 es01
docker rm kib01 es01
docker network rm elastic
```

> Warning: This deletes all your data stored in Elasticsearch.

---

## Troubleshooting Guide

### Problem 1: `docker: command not found`

**Cause:** Docker is not installed or not in your PATH.

**Fix:**
1. Open Docker Desktop on Windows
2. In Docker Desktop Settings → Resources → WSL Integration, make sure your Ubuntu distro is enabled
3. Close and reopen your WSL terminal
4. Try `docker --version` again

---

### Problem 2: Elasticsearch starts but then stops

**Check logs:**
```bash
docker logs es01 | tail -50
```

**Most common cause on WSL2:** `vm.max_map_count` is too low.

**Fix (run this in WSL):**
```bash
sudo sysctl -w vm.max_map_count=262144
```

**Make it permanent** (survives reboots):
```bash
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

**Why this matters:** Elasticsearch uses memory-mapped files for performance. Linux defaults (`65530`) are too low. `262144` is the minimum Elasticsearch requires.

---

### Problem 3: Kibana keeps restarting or exits immediately

**Check logs:**
```bash
docker logs kib01 | tail -50
```

**Common causes and fixes:**

| Error in logs | Fix |
|---|---|
| `Unable to retrieve version from Elasticsearch` | Elasticsearch is not ready yet. Wait 30s and restart: `docker restart kib01` |
| `Hostname/IP does not match certificate` | Add `-e ELASTICSEARCH_SSL_VERIFICATIONMODE=none` to your Kibana docker run command |
| `Connection refused` | Elasticsearch container is stopped. Run `docker start es01` first |
| `Authentication error` | Wrong `kibana_system` password. Reset it (Step 4b) and recreate Kibana container |

---

### Problem 4: `curl` to Elasticsearch returns SSL error

**Symptom:**
```
curl: (60) SSL certificate problem: unable to get local issuer certificate
```

**Fix:** Always use `--cacert ~/ELK/http_ca.crt` in your curl command:
```bash
curl --cacert ~/ELK/http_ca.crt -u elastic:"yourpassword" https://localhost:9200
```

---

### Problem 5: Kibana shows "Kibana server is not ready yet"

This is normal for the first 30–60 seconds after starting Kibana. Just wait and refresh.

If it persists after 2 minutes:
```bash
docker logs kib01 2>&1 | grep -E "ERROR|error|failed"
```

---

### Problem 6: Port 9200 or 5601 already in use

```bash
# Find what is using the port
sudo lsof -i :9200
sudo lsof -i :5601
```

Stop the conflicting process, then start the containers again.

---

### Problem 7: `network elastic already exists`

```bash
docker network ls | grep elastic
```

If it exists, just use it. If you want a clean start:
```bash
docker network rm elastic
docker network create elastic
```

---

### Problem 8: "Kibana has not been configured" page appears

This happens when Kibana starts without connection settings. Make sure you used all the `-e` environment variable flags in Step 5. The most important ones:
- `ELASTICSEARCH_HOSTS`
- `ELASTICSEARCH_USERNAME`
- `ELASTICSEARCH_PASSWORD`

---

## Useful Commands Cheat Sheet

```bash
# See all running containers
docker ps

# See ALL containers (including stopped)
docker ps -a

# See logs from a container
docker logs es01
docker logs kib01

# Follow logs in real time (like tail -f)
docker logs -f kib01

# Check cluster health
curl --cacert ~/ELK/http_ca.crt -u elastic:"yourpassword" https://localhost:9200/_cluster/health

# See all indices
curl --cacert ~/ELK/http_ca.crt -u elastic:"yourpassword" https://localhost:9200/_cat/indices?v

# Get inside the Elasticsearch container (like SSH)
docker exec -it es01 bash

# Restart a container
docker restart kib01

# Remove a stopped container
docker rm kib01
```

---

## WSL2 Specific Notes

### Accessing Kibana from Windows browser
Even though Kibana runs in WSL2, you can access it from your Windows browser at:
```
http://localhost:5601
```
This works because Docker Desktop bridges the ports between WSL2 and Windows.

### Memory Settings
WSL2 can limit RAM available to Linux. If Elasticsearch is slow or crashing, add this to `C:\Users\<YourUser>\.wslconfig` on Windows:

```ini
[wsl2]
memory=4GB
processors=2
```

Then restart WSL: open PowerShell and run `wsl --shutdown`, then reopen your terminal.

### Clock Drift Warning
You may see this in Elasticsearch logs:
```
absolute clock went backwards by [1.4s]
```
This is a known WSL2 issue — the WSL2 clock drifts when Windows suspends/resumes. It is a **warning only** and does not affect functionality.

---

## Current Setup Reference

After following this guide, your setup looks like this:

```
Windows Browser
      |
      | http://localhost:5601
      v
 +-----------+
 |  kib01    |  (Kibana container, port 5601)
 +-----------+
      |
      | https://es01:9200 (via Docker "elastic" network)
      v
 +-----------+
 |  es01     |  (Elasticsearch container, port 9200)
 +-----------+
      |
      | https://localhost:9200
      v
 Your terminal / curl / apps
```

---

*Version: 9.3.2 | Platform: WSL2 + Docker Desktop | Single Node Setup*
