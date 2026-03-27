# Integrating a Docker App with ELK Stack — Log Shipping & Kibana Views

> **Goal:** Take any Docker app and get its logs flowing into Elasticsearch, then search and visualise them in Kibana.

---

## How It All Works — The Big Picture

Before writing a single command, understand the journey a log message takes:

```
[ Flask App ]
     | writes JSON log to stdout
     v
[ Docker Engine ]
     | stores log as a file under /var/lib/docker/containers/
     v
[ Filebeat ]
     | watches Docker container logs
     | reads the file, attaches container metadata (name, image, labels)
     v
[ Logstash ]
     | parses the JSON log
     | renames/promotes fields for easier Kibana querying
     | adds a "service" tag
     v
[ Elasticsearch (es01) ]
     | stores each log as a searchable document
     | in index: flask-app-YYYY.MM.DD
     v
[ Kibana (kib01) ]
     | you search, filter, and visualise here
```

Think of it like a postal system:
- **Filebeat** = the postman who picks up letters (logs) from each house (container)
- **Logstash** = the sorting office that reads, stamps, and routes each letter
- **Elasticsearch** = the giant filing cabinet where all letters are stored and indexed
- **Kibana** = the reading room where you can find and read any letter instantly

---

## Prerequisites

- ELK Stack running (Elasticsearch on port `9200`, Kibana on port `5601`)
- Both running on Docker network `elastic`
- CA certificate at `~/ELK/http_ca.crt`

**Verify your ELK stack is up before starting:**
```bash
docker ps | grep -E "es01|kib01"
```
Both should show `Up`.

---

## Project Structure

After integration, the demo app folder looks like this:

```
demo-app/
├── app.py                        ← Flask app with JSON structured logging
├── Dockerfile
├── requirements.txt
├── docker-compose.yml            ← defines app, logstash, filebeat
├── .env                          ← holds ELASTIC_PASSWORD (do not commit to git)
└── elk/
    ├── certs/
    │   └── http_ca.crt           ← CA cert copied from es01 (for HTTPS trust)
    ├── filebeat/
    │   └── filebeat.yml          ← tells Filebeat which containers to watch
    └── logstash/
        └── pipeline/
            └── logstash.conf     ← parses logs and forwards to Elasticsearch
```

---

## Step 1: Copy the CA Certificate into Your App

Logstash needs to trust the HTTPS connection to `es01`. The certificate lives inside the `es01` container — copy it into your project.

```bash
# Create the certs folder inside your app
mkdir -p ~/git-repos/demo-docker-app/demo-app/elk/certs

# Copy the cert from the running es01 container
docker cp es01:/usr/share/elasticsearch/config/certs/http_ca.crt \
  ~/git-repos/demo-docker-app/demo-app/elk/certs/http_ca.crt
```

**What this does:**
- `docker cp` = copy a file out of a container to your local machine
- This cert file proves to Logstash that `es01` is who it says it is (prevents man-in-the-middle attacks)

---

## Step 2: Create the `.env` File

This file holds the `elastic` user password. Docker Compose reads it automatically.

```bash
# Create .env in your app directory
cat > ~/git-repos/demo-docker-app/demo-app/.env << 'EOF'
# Password for the elastic superuser in your running ELK stack
# Logstash uses this to authenticate when sending logs to es01
ELASTIC_PASSWORD=&lt;your-elastic-password&gt;
EOF
```

> **Important:** Add `.env` to your `.gitignore` — never commit passwords to git.
> ```bash
> echo ".env" >> .gitignore
> ```

**Why a separate file?**
Instead of hardcoding the password inside `docker-compose.yml` (which you commit to git), we put it in `.env` which stays local to your machine only.

---

## Step 3: The `docker-compose.yml` — Explained

This file starts three services: the Flask app, Logstash, and Filebeat.

> Elasticsearch and Kibana are **NOT** here — they are already running as `es01` and `kib01`. We connect to them via the external `elastic` Docker network.

```yaml
services:

  # ── Flask Application ──────────────────────────────────────────────────────
  app:
    build: .
    ports:
      - "5000:5000"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    labels:
      # These labels are the "opt-in" signal to Filebeat:
      # "yes, please collect logs from this container"
      - "co.elastic.logs/enabled=true"
      - "co.elastic.logs/json.keys_under_root=true"
      - "co.elastic.logs/json.overwrite_keys=true"
    networks:
      - elk

  # ── Logstash ───────────────────────────────────────────────────────────────
  logstash:
    image: docker.elastic.co/logstash/logstash:9.3.2
    volumes:
      - ./elk/logstash/pipeline:/usr/share/logstash/pipeline:ro
      # Mount the CA cert so Logstash can verify the HTTPS connection to es01
      - ./elk/certs/http_ca.crt:/usr/share/logstash/certs/http_ca.crt:ro
    ports:
      - "5044:5044"
    environment:
      - LS_JAVA_OPTS=-Xms256m -Xmx256m
      - xpack.monitoring.enabled=false
      # Inject the password from .env into the container
      # Logstash pipeline uses ${ELASTIC_PASSWORD} to read this value
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
    networks:
      - elk      # receives logs from Filebeat
      - elastic  # sends logs to es01

  # ── Filebeat ───────────────────────────────────────────────────────────────
  filebeat:
    image: docker.elastic.co/beats/filebeat:9.3.2
    user: root
    volumes:
      - ./elk/filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    depends_on:
      - logstash
    networks:
      - elk

networks:
  elk:
    driver: bridge    # internal: app, filebeat, logstash talk here

  elastic:
    external: true    # external: already exists from ELK stack setup
```

**Key concepts:**

| Concept | Explanation |
|---|---|
| `labels: co.elastic.logs/enabled=true` | Tells Filebeat "watch this container". Without this label, Filebeat ignores the container |
| `networks: [elk, elastic]` | Logstash is on both networks — `elk` to receive from Filebeat, `elastic` to send to `es01` |
| `external: true` | This network already exists. Docker Compose joins it rather than creating a new one |
| `- ELASTIC_PASSWORD=${ELASTIC_PASSWORD}` | Reads the value from `.env` and injects it into the container as an environment variable |

---

## Step 4: The `filebeat.yml` — Explained

```yaml
filebeat.autodiscover:
  providers:
    - type: docker
      hints.enabled: true
      templates:
        - condition:
            equals:
              # Only collect from containers that have this label
              docker.container.labels.co.elastic.logs/enabled: "true"
          config:
            - type: container
              paths:
                - /var/lib/docker/containers/${data.docker.container.id}/*.log
              json.keys_under_root: true
              json.overwrite_keys: true
              json.message_key: log

processors:
  - add_docker_metadata:
      host: "unix:///var/run/docker.sock"
  - add_host_metadata: ~

output.logstash:
  hosts: ["logstash:5044"]
```

**What each part does:**

| Section | What it does |
|---|---|
| `autodiscover.providers: docker` | Automatically discovers new containers as they start — no manual config needed per container |
| `hints.enabled` | Reads `co.elastic.logs/*` labels from each container to decide what/how to collect |
| `json.keys_under_root` | Moves JSON fields to the top level of the log event (not nested under "message") |
| `add_docker_metadata` | Attaches container name, image, labels to every log event |
| `output.logstash` | Sends collected logs to Logstash on port 5044 |

---

## Step 5: The `logstash.conf` — Explained

```
input {
  beats {
    port => 5044       # receive from Filebeat
  }
}

filter {
  if [message] =~ /^\s*\{/ {    # only process JSON log lines
    json {
      source => "message"
      target => "log"
    }
    mutate {
      rename => {
        "[log][levelname]"   => "level"       # INFO, WARNING, ERROR
        "[log][message]"     => "log_message" # the actual text
        "[log][asctime]"     => "timestamp"
        "[log][endpoint]"    => "endpoint"    # /health, /
        "[log][method]"      => "http_method" # GET, POST
        "[log][remote_addr]" => "client_ip"
      }
    }
  }
  mutate {
    add_field => { "service" => "flask-demo-app" }
  }
}

output {
  elasticsearch {
    hosts    => ["https://es01:9200"]
    user     => "elastic"
    password => "${ELASTIC_PASSWORD}"         # reads from env var
    ssl_enabled                 => true
    ssl_certificate_authorities => ["/usr/share/logstash/certs/http_ca.crt"]
    ssl_verification_mode       => "none"
    index    => "flask-app-%{+YYYY.MM.dd}"   # daily index e.g. flask-app-2026.03.27
  }
}
```

**Why rename the fields?**
The raw Flask log looks like `{"levelname": "INFO", "message": "Health check", ...}`. Renaming `levelname` → `level`, `message` → `log_message` etc. makes the fields shorter, cleaner, and easier to find in Kibana's dropdown menus.

**Why a daily index (`flask-app-2026.03.27`)?**
Daily indices let you easily delete old data (just delete yesterday's index) and keep searches fast by not querying all data at once.

---

## Step 6: Start the App Stack

```bash
cd ~/git-repos/demo-docker-app/demo-app

docker compose up -d --build
```

**What this does:**
- `--build` = rebuild the Flask app image (picks up any code changes)
- `-d` = run in background

**Verify all services started:**
```bash
docker ps
```

You should see 5 containers running:
```
NAMES                  STATUS
demo-app-filebeat-1    Up
demo-app-logstash-1    Up
demo-app-app-1         Up
kib01                  Up
es01                   Up
```

**Check Logstash connected to Elasticsearch:**
```bash
docker logs demo-app-logstash-1 | grep -E "pipeline|Elasticsearch"
```

Look for: `Pipelines running` and `Elasticsearch version determined`

---

## Step 7: Generate Logs

Hit the Flask app endpoints to produce log entries:

```bash
# Single requests
curl http://localhost:5000/
curl http://localhost:5000/health

# Generate a burst of traffic (10 requests to each endpoint)
for i in {1..10}; do
  curl -s http://localhost:5000/ > /dev/null
  curl -s http://localhost:5000/health > /dev/null
done
echo "Done — 20 requests sent"
```

---

## Step 8: Verify Logs Reached Elasticsearch

```bash
curl --cacert ~/ELK/http_ca.crt \
  -u "elastic:&lt;your-elastic-password&gt;" \
  "https://localhost:9200/_cat/indices/flask-app-*?v"
```

**Expected output:**
```
health status index                docs.count
yellow open   flask-app-2026.03.27       1911
```

If you see `docs.count > 0`, logs are flowing correctly.

---

## Step 9: View Logs in Kibana

### 9a. Create a Data View

A Data View tells Kibana which Elasticsearch index to read from.

1. Open `http://localhost:5601` and login with `elastic` / `&lt;your-elastic-password&gt;`
2. Click the hamburger menu (top-left) → **Management** → **Stack Management**
3. Under **Kibana**, click **Data Views**
4. Click **Create data view**
5. Fill in:
   - **Name:** `Flask App Logs`
   - **Index pattern:** `flask-app-*` (the `*` matches all daily indices)
   - **Timestamp field:** `@timestamp`
6. Click **Save data view to Kibana**

### 9b. Search Logs in Discover

1. Click hamburger menu → **Discover**
2. In the top-left dropdown, select **Flask App Logs**
3. Set the time range (top-right) to **Last 1 hour**
4. You will see all your log events as rows

**Useful fields to add as columns:**
- Click `level` in the left field list → **+** to add as column
- Click `log_message` → **+**
- Click `endpoint` → **+**
- Click `http_method` → **+**

Now each row shows: timestamp | level | message | endpoint | method

---

## Kibana Search Query Examples

Use these in the search bar at the top of Discover.

### KQL (Kibana Query Language) — beginner friendly

```
# All logs from the health endpoint
endpoint: "/health"

# Only ERROR level logs
level: "ERROR"

# All INFO logs on the home endpoint
level: "INFO" and endpoint: "/"

# All logs from a specific service
service: "flask-demo-app"

# Logs containing a specific word in the message
log_message: "Health check"

# Logs NOT from health endpoint
NOT endpoint: "/health"
```

### Lucene Syntax — more powerful

```
# Wildcard search on message
log_message:Health*

# Range — not typically used for logs but useful for numeric fields
# (e.g. find logs from a specific time range via filters instead)

# Exact phrase match
log_message:"Incoming request"
```

### Time-based filtering
Use the time picker (top-right) to narrow down:
- **Quick:** Last 15 minutes / Last 1 hour / Today
- **Absolute:** Pick exact start and end dates
- **Relative:** "Now minus 2 hours"

---

## Step 10: Create a Simple Dashboard

### Add a Log Count Over Time chart

1. Click hamburger → **Dashboard** → **Create dashboard**
2. Click **Create visualization**
3. Select **Lens**
4. Drag **@timestamp** to the x-axis
5. Leave the y-axis as **Count of records**
6. Change the chart type to **Bar vertical stacked**
7. Click **Save and return**

### Add a Log Levels Breakdown (Pie chart)

1. Click **Create visualization** again → **Lens**
2. Change chart type to **Pie**
3. Drag **level** field to the **Slice by** section
4. Click **Save and return**

### Save the Dashboard

1. Click **Save** (top-right)
2. Name it: `Flask App Overview`

Now you have a live dashboard that updates as new logs arrive.

---

## Stopping and Restarting

### Stop only the app stack (keep ELK running)
```bash
cd ~/git-repos/demo-docker-app/demo-app
docker compose down
```

### Start everything back up
```bash
# Start ELK first
docker start es01
sleep 20
docker start kib01

# Then start the app
cd ~/git-repos/demo-docker-app/demo-app
docker compose up -d
```

### Restart just Logstash (e.g. after a config change)
```bash
cd ~/git-repos/demo-docker-app/demo-app
docker compose restart logstash
```

---

## Troubleshooting

### Logs not appearing in Kibana

**Step 1 — Check the index exists:**
```bash
curl --cacert ~/ELK/http_ca.crt -u "elastic:&lt;your-elastic-password&gt;" \
  "https://localhost:9200/_cat/indices/flask-app-*?v"
```
If no index → logs never reached Elasticsearch. Check Logstash.

**Step 2 — Check Logstash is running:**
```bash
docker logs demo-app-logstash-1 | tail -20
```
Look for: `Pipelines running` = good. `ERROR` or `401` = credentials or network issue.

**Step 3 — Check Filebeat is running:**
```bash
docker logs demo-app-filebeat-1 | tail -20
```
Look for: `Connection to backoff(logstash...)` established.

**Step 4 — Check the Flask app has the right labels:**
```bash
docker inspect demo-app-app-1 | grep -A5 "Labels"
```
Must include `co.elastic.logs/enabled: true`.

---

### Logstash error: `401 Unauthorized`

The `elastic` password in `.env` doesn't match what Elasticsearch expects.

**Fix — reset the password:**
```bash
echo "y" | docker exec -i es01 \
  /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic -s
```
Update `.env` with the new value, then restart Logstash:
```bash
docker compose up -d --force-recreate logstash
```

> **Tip:** Avoid special characters (`*`, `+`, `=`) in the password — use the API to set a clean alphanumeric one:
> ```bash
> curl --cacert ~/ELK/http_ca.crt \
>   -u "elastic:CURRENT_PASSWORD" \
>   -X POST "https://localhost:9200/_security/user/elastic/_password" \
>   -H 'Content-Type: application/json' \
>   -d '{"password":"YourNewCleanPassword123"}'
> ```

---

### `network elastic not found`

The external `elastic` network doesn't exist yet — ELK stack isn't running.

**Fix:**
```bash
docker start es01 kib01
```

---

### Kibana shows no data in Discover

1. Make sure the time range (top-right) covers when you generated logs
2. Confirm the Data View index pattern is `flask-app-*`
3. Run the index check: `curl ... /_cat/indices/flask-app-*?v`

---

## Current Credentials Reference

| Service | URL | Username | Password |
|---|---|---|---|
| Kibana | `http://localhost:5601` | `elastic` | `&lt;your-elastic-password&gt;` |
| Elasticsearch | `https://localhost:9200` | `elastic` | `&lt;your-elastic-password&gt;` |
| Flask App | `http://localhost:5000` | — | — |

---

*For ELK Stack installation, see [elk-single-node-docker-wsl2-setup.md](./elk-single-node-docker-wsl2-setup.md)*
*For complete teardown, see [elk-stack-complete-teardown.md](./elk-stack-complete-teardown.md)*
