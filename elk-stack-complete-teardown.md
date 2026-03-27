# Complete ELK Stack Uninstall Guide

> This guide removes **everything** — containers, images, networks, volumes, certificates, and folders. Follow only the sections you need. Steps are ordered from "least destructive" to "completely wipe everything."

---

## Before You Start — Check What's Running

Always look before you delete. Run this first:

```bash
# See what containers are running
docker ps -a | grep -E "es01|kib01"

# See what networks exist
docker network ls | grep elastic

# See what images are downloaded
docker images | grep elastic

# See what volumes exist
docker volume ls | grep elastic
```

This gives you a clear picture of what's on your machine before removing anything.

---

## Level 1: Stop the Services (Reversible)

> Use this when you want to free up memory but keep everything for later.

```bash
docker stop kib01 es01
```

**What this does:**
- `docker stop` = sends a graceful shutdown signal (like clicking "Quit" on an app)
- Stops Kibana first (always stop Kibana before Elasticsearch)
- Your data inside the containers is **completely safe** — nothing is deleted

**To bring it back:**
```bash
docker start es01
sleep 20
docker start kib01
```

---

## Level 2: Remove the Containers (Data stays in volumes if any)

> Use this when you want to delete the containers but you used Docker volumes for data storage.

First, stop them:
```bash
docker stop kib01 es01
```

Then remove the containers:
```bash
docker rm kib01 es01
```

**What this does:**
- `docker rm` = deletes the container (the running instance)
- The Docker **image** is still on your machine (you don't need to re-download)
- If you did **not** use a `-v` volume flag when starting, all Elasticsearch data is gone too

**Verify they are gone:**
```bash
docker ps -a | grep -E "es01|kib01"
```

Should return nothing.

---

## Level 3: Remove the Docker Network

```bash
docker network rm elastic
```

**What this does:**
- Removes the private network we created that allowed `es01` and `kib01` to talk to each other
- Safe to do once both containers are removed

**Verify:**
```bash
docker network ls | grep elastic
```

Should return nothing.

---

## Level 4: Remove the Docker Images (Free up disk space)

> This frees up the most disk space (~1.5 GB total). You will need to re-download if you want to run ELK again.

```bash
docker rmi docker.elastic.co/elasticsearch/elasticsearch:9.3.2
docker rmi docker.elastic.co/kibana/kibana:9.3.2
```

**What this does:**
- `docker rmi` = remove image (the downloaded software package)
- After this, running `docker run ...` for ELK will need to re-download the images
- This does NOT affect any other Docker images on your machine

**Verify:**
```bash
docker images | grep elastic
```

Should return nothing.

---

## Level 5: Remove Certificates and Local Files

```bash
rm -rf ~/ELK
```

**What this does:**
- Deletes the `~/ELK/` folder we created to store the `http_ca.crt` certificate
- This is just a copy — the original lives inside the Elasticsearch container
- Safe to delete; you can always copy it again if you reinstall

**Verify:**
```bash
ls ~/ELK
```

Should say: `No such file or directory`

---

## Level 6: Remove Docker Volumes (if you created any)

If you started Elasticsearch with a named volume (e.g., `-v esdata:/usr/share/elasticsearch/data`), the data lives in a Docker volume that persists even after `docker rm`.

**List all volumes:**
```bash
docker volume ls
```

**Remove a specific volume:**
```bash
docker volume rm esdata
```

**Remove ALL unused volumes at once:**
```bash
docker volume prune
```

> Warning: `docker volume prune` removes ALL volumes not attached to a running container. Only run this if you are sure.

---

## Complete Uninstall — All-in-One Commands

Run these commands **in order** to remove everything:

```bash
# 1. Stop containers
docker stop kib01 es01

# 2. Remove containers
docker rm kib01 es01

# 3. Remove network
docker network rm elastic

# 4. Remove images
docker rmi docker.elastic.co/elasticsearch/elasticsearch:9.3.2
docker rmi docker.elastic.co/kibana/kibana:9.3.2

# 5. Remove local cert folder
rm -rf ~/ELK

# 6. Remove any ELK volumes (only if you created named volumes)
docker volume prune -f
```

**Verify everything is gone:**
```bash
echo "=== Containers ===" && docker ps -a | grep -E "es01|kib01" || echo "None"
echo "=== Networks ===" && docker network ls | grep elastic || echo "None"
echo "=== Images ===" && docker images | grep elastic || echo "None"
echo "=== Volumes ===" && docker volume ls | grep -E "elastic|esdata" || echo "None"
echo "=== Local files ===" && ls ~/ELK 2>/dev/null || echo "None"
```

All five sections should print "None".

---

## Removing Docker Itself (Optional — Nuclear Option)

> Only do this if you want to completely remove Docker from WSL2.

```bash
sudo apt remove docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo apt autoremove -y
```

**Also remove Docker's data directory:**
```bash
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
```

**Remove Docker's apt repository:**
```bash
sudo rm /etc/apt/sources.list.d/docker.list
sudo rm /etc/apt/keyrings/docker.gpg
```

> After this, Docker is completely gone from your WSL2 instance. To use it again you must reinstall Docker Desktop and re-enable WSL2 integration.

---

## Quick Reference — What Each Command Removes

| Command | What it removes | Data lost? |
|---|---|---|
| `docker stop es01 kib01` | Stops the running process | No |
| `docker rm es01 kib01` | Deletes containers | Yes (if no volume) |
| `docker network rm elastic` | Deletes the private network | No |
| `docker rmi elasticsearch:9.3.2` | Deletes the downloaded image (~1 GB) | No |
| `docker rmi kibana:9.3.2` | Deletes the downloaded image (~500 MB) | No |
| `rm -rf ~/ELK` | Deletes local cert copy | No (cert is in container) |
| `docker volume prune` | Deletes ALL unused volumes | Yes |
| `sudo apt remove docker-ce ...` | Removes Docker from WSL2 | Yes — everything |

---

*For reinstallation, follow the steps in [README.md](./README.md).*
