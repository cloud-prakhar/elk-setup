# Linux & Windows Server Monitoring with ELK — System-Level Logging

> **Goal:** Ship operating-system logs (Linux `syslog`/`auth.log`/journald and Windows Event Logs) from real servers into the ELK stack you already built on WSL2 + Docker, then search them in Kibana.

This guide assumes the stack from [elk-single-node-docker-wsl2-setup.md](./elk-single-node-docker-wsl2-setup.md) is already running (`es01` + `kib01` on the `elastic` Docker network). Nothing about that setup changes — we only add a **Logstash receiver** container and install **Filebeat agents** on the servers you want to monitor.

---

## How It All Works — The Big Picture

Application logging (see [elk-app-log-integration.md](./elk-app-log-integration.md)) ran Filebeat *inside* Docker to watch container log files. System monitoring is different: **Filebeat runs directly on the server's operating system**, as a service, reading OS log files and event logs.

```
[ Linux Server ]                        [ Windows Server ]
   /var/log/syslog                        Windows Event Log
   /var/log/auth.log                        - Security
   journald                                 - System
        |                                    - Application
        v                                         |
[ Filebeat (systemd service) ]          [ Filebeat (Windows service) ]
        |                                         |
        |  TCP 5044 (Beats protocol)              |
        +--------------------+--------------------+
                             v
                  [ Logstash — logstash-sys container ]
                     | parses syslog lines with grok
                     | normalises Windows event fields
                     | routes to the right index
                     v
                  [ Elasticsearch (es01) ]
                     | linux-system-YYYY.MM.dd
                     | windows-system-YYYY.MM.dd
                     v
                  [ Kibana (kib01) ]
```

Same postal analogy as before, but now the postman is stationed **on each server** instead of inside Docker:

- **Filebeat** = an agent installed on every server, tailing OS log sources
- **Logstash** = one central sorting office that all servers ship to
- **Elasticsearch** = the filing cabinet
- **Kibana** = the reading room

> **Note on scope:** Filebeat ships *logs* (text/events). It does **not** ship metrics like CPU %, memory, or disk usage — that is Metricbeat's job. See the [Optional: Metrics](#optional-adding-metrics-with-metricbeat) section at the end.

---

## Prerequisites

### 1. The ELK stack must be running

```bash
docker ps | grep -E "es01|kib01"
```
Both must show `Up`. If not:
```bash
docker start es01
sleep 20
docker start kib01
```

### 2. Files and values you need in hand

| Item | Where it comes from | Used by |
|---|---|---|
| `~/ELK/http_ca.crt` | Copied from `es01` during setup (Step 4) | Logstash → Elasticsearch TLS |
| `elastic` password | Set during setup (Step 5) | Logstash authentication |
| Docker network `elastic` | Created during setup (Step 2) | Logstash → `es01` connectivity |
| Port `5044` reachable from your servers | Configured below | Filebeat → Logstash |

Verify the cert and password work before going further:
```bash
curl --cacert ~/ELK/http_ca.crt \
  -u "elastic:<your-elastic-password>" \
  "https://localhost:9200/_cluster/health?pretty"
```
Expect `"status": "green"` or `"yellow"` (yellow is normal on a single node).

### 3. Working folder on WSL2

```bash
mkdir -p ~/ELK/logstash-sys/pipeline
mkdir -p ~/ELK/logstash-sys/certs
cp ~/ELK/http_ca.crt ~/ELK/logstash-sys/certs/http_ca.crt
```

### 4. Network reachability checklist

This is the part people get wrong most often. Where your monitored servers live decides how much networking you need:

| Monitored server | What you need |
|---|---|
| WSL2 itself (the same Ubuntu you run Docker on) | Nothing extra — Filebeat connects to `localhost:5044` |
| Another WSL2 distro on the same PC | Use the WSL2 IP (`hostname -I`) |
| **The Windows host machine** | Windows can reach WSL2 via `localhost:5044` on recent Windows builds. If not, add a `portproxy` rule (see below) |
| A different physical/virtual server on the LAN | Requires Windows port forwarding **and** a firewall rule (see below) |

**Windows port forwarding into WSL2** (run in an **elevated PowerShell** on the Windows host):

```powershell
# Find the WSL2 IP address
wsl hostname -I
# e.g. 172.24.118.5

# Forward Windows port 5044 -> WSL2 port 5044
netsh interface portproxy add v4tov4 `
  listenaddress=0.0.0.0 listenport=5044 `
  connectaddress=172.24.118.5 connectport=5044

# Allow it through Windows Firewall
New-NetFirewallRule -DisplayName "ELK Logstash Beats 5044" `
  -Direction Inbound -Protocol TCP -LocalPort 5044 -Action Allow

# Verify
netsh interface portproxy show v4tov4
```

> **Important:** The WSL2 IP **changes on every reboot**. Re-run the `portproxy` command after restarting Windows, or enable *mirrored networking mode* in `%USERPROFILE%\.wslconfig`:
> ```ini
> [wsl2]
> networkingMode=mirrored
> ```
> (Requires Windows 11 22H2+; with mirrored mode, `localhost:5044` just works and no portproxy is needed.)

### 5. Permissions

- **Linux:** Filebeat must run as `root` — `/var/log/auth.log` and journald are not world-readable.
- **Windows:** Filebeat must run as a service under `LocalSystem` (the default from the install script) to read the **Security** event channel.

---

## Part A — Set Up the Central Logstash Receiver

One Logstash container receives from **all** servers, Linux and Windows alike.

### A1. Create the pipeline config

```bash
nano ~/ELK/logstash-sys/pipeline/logstash.conf
```

Paste this — the full, annotated config:

```
# ─────────────────────────────────────────────────────────────────────────────
# INPUT — one Beats listener for every server that ships to us
# ─────────────────────────────────────────────────────────────────────────────
input {
  beats {
    port => 5044
  }
}

# ─────────────────────────────────────────────────────────────────────────────
# FILTER — branch on the tag each Filebeat agent stamps on its events
# ─────────────────────────────────────────────────────────────────────────────
filter {

  # ── LINUX ──────────────────────────────────────────────────────────────────
  if "linux-system" in [tags] {

    # Classic syslog line looks like:
    # Aug  9 06:14:02 web01 sshd[2311]: Accepted password for prakhar from 10.0.0.5
    grok {
      match => {
        "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:process}(?:\[%{POSINT:pid:int}\])?: %{GREEDYDATA:log_message}"
      }
      # If the line does not match, tag it instead of dropping it
      tag_on_failure => ["_syslog_grok_failure"]
    }

    # syslog timestamps have no year — Logstash assumes the current year
    date {
      match  => [ "syslog_timestamp", "MMM  d HH:mm:ss", "MMM d HH:mm:ss" ]
      target => "@timestamp"
      timezone => "Asia/Kolkata"          # set to YOUR server's timezone
    }

    # Classify security-relevant auth events so they are easy to filter later
    if [log][file][path] == "/var/log/auth.log" or [process] in ["sshd", "sudo", "su"] {
      mutate { add_field => { "event_category" => "authentication" } }
    }

    if [log_message] =~ /Failed password|authentication failure|Invalid user/ {
      mutate {
        add_field => { "event_outcome" => "failure" }
        add_tag   => [ "auth_failure" ]
      }
    } else if [log_message] =~ /Accepted password|Accepted publickey|session opened/ {
      mutate { add_field => { "event_outcome" => "success" } }
    }

    # Pull the source IP out of SSH messages when present
    grok {
      match => { "log_message" => "from %{IP:source_ip}" }
      tag_on_failure => []                # silent — most lines have no IP
    }

    mutate {
      add_field    => { "os_family" => "linux" }
      remove_field => [ "syslog_timestamp" ]
    }
  }

  # ── WINDOWS ────────────────────────────────────────────────────────────────
  # Filebeat's winlog input already delivers structured fields
  # (winlog.event_id, winlog.channel, winlog.provider_name, ...).
  # We only promote the useful ones to short top-level names.
  if "windows-system" in [tags] {

    mutate {
      rename => {
        "[winlog][event_id]"      => "event_id"
        "[winlog][channel]"       => "event_channel"
        "[winlog][provider_name]" => "event_provider"
        "[winlog][computer_name]" => "computer_name"
        "[winlog][record_id]"     => "event_record_id"
        "[message]"               => "log_message"
      }
      add_field => { "os_family" => "windows" }
    }

    # Map the noisy numeric event IDs that actually matter
    if [event_id] == 4625 {
      mutate {
        add_field => { "event_category" => "authentication" }
        add_field => { "event_outcome"  => "failure" }
        add_field => { "event_name"     => "Failed logon" }
        add_tag   => [ "auth_failure" ]
      }
    } else if [event_id] == 4624 {
      mutate {
        add_field => { "event_category" => "authentication" }
        add_field => { "event_outcome"  => "success" }
        add_field => { "event_name"     => "Successful logon" }
      }
    } else if [event_id] == 4720 {
      mutate { add_field => { "event_name" => "User account created" } }
    } else if [event_id] == 7036 {
      mutate { add_field => { "event_name" => "Service state changed" } }
    } else if [event_id] == 1074 or [event_id] == 6006 or [event_id] == 6008 {
      mutate { add_field => { "event_name" => "Shutdown / restart" } }
    }
  }

  # ── COMMON ─────────────────────────────────────────────────────────────────
  # Normalise the hostname so one Kibana field works for both OS families
  if [host][name] {
    mutate { add_field => { "server_name" => "%{[host][name]}" } }
  }
}

# ─────────────────────────────────────────────────────────────────────────────
# OUTPUT — one index per OS family, rotated daily
# ─────────────────────────────────────────────────────────────────────────────
output {
  if "linux-system" in [tags] {
    elasticsearch {
      hosts                       => ["https://es01:9200"]
      user                        => "elastic"
      password                    => "${ELASTIC_PASSWORD}"
      ssl_enabled                 => true
      ssl_certificate_authorities => ["/usr/share/logstash/certs/http_ca.crt"]
      ssl_verification_mode       => "none"
      index                       => "linux-system-%{+YYYY.MM.dd}"
    }
  }
  else if "windows-system" in [tags] {
    elasticsearch {
      hosts                       => ["https://es01:9200"]
      user                        => "elastic"
      password                    => "${ELASTIC_PASSWORD}"
      ssl_enabled                 => true
      ssl_certificate_authorities => ["/usr/share/logstash/certs/http_ca.crt"]
      ssl_verification_mode       => "none"
      index                       => "windows-system-%{+YYYY.MM.dd}"
    }
  }
  else {
    # Anything that arrives without a known tag — so nothing is silently lost
    elasticsearch {
      hosts                       => ["https://es01:9200"]
      user                        => "elastic"
      password                    => "${ELASTIC_PASSWORD}"
      ssl_enabled                 => true
      ssl_certificate_authorities => ["/usr/share/logstash/certs/http_ca.crt"]
      ssl_verification_mode       => "none"
      index                       => "unclassified-%{+YYYY.MM.dd}"
    }
  }
}
```

**Why the config is shaped this way:**

| Choice | Reason |
|---|---|
| Branch on `tags`, not on hostname | You can add 50 servers without touching Logstash again — the agent declares what it is |
| `grok` for Linux, `rename` for Windows | Linux syslog is a raw text line and must be parsed; the Windows event log is already structured |
| `date` filter with explicit `timezone` | syslog timestamps carry no year and no offset — without this, events land at the wrong time in Kibana |
| `tag_on_failure` instead of `drop` | An unparsed line still reaches Elasticsearch; you can find it later with `tags: "_syslog_grok_failure"` |
| Daily index per OS | Delete old data by dropping one index; keeps searches narrow |
| `ssl_verification_mode => "none"` | The dev cert from `es01` has no matching SAN for every client. Fine for a lab — use `full` with a proper cert in production |

### A2. Start the Logstash container

```bash
docker run -d \
  --name logstash-sys \
  --net elastic \
  -p 5044:5044 \
  -e LS_JAVA_OPTS="-Xms512m -Xmx512m" \
  -e xpack.monitoring.enabled=false \
  -e ELASTIC_PASSWORD="<your-elastic-password>" \
  -v ~/ELK/logstash-sys/pipeline:/usr/share/logstash/pipeline:ro \
  -v ~/ELK/logstash-sys/certs/http_ca.crt:/usr/share/logstash/certs/http_ca.crt:ro \
  docker.elastic.co/logstash/logstash:9.3.2
```

| Flag | What it does |
|---|---|
| `--net elastic` | Joins the existing ELK network so `https://es01:9200` resolves by name |
| `-p 5044:5044` | Exposes the Beats listener to WSL2 (and then to Windows/LAN via portproxy) |
| `-e ELASTIC_PASSWORD` | Read by `${ELASTIC_PASSWORD}` in the pipeline. Prefer `--env-file ~/ELK/.env` so the password stays out of shell history |
| `-v .../pipeline` | Mounts your config read-only; edit on the host, restart the container to apply |

**Verify it started cleanly:**
```bash
docker logs logstash-sys 2>&1 | grep -E "Pipeline started|Beats inputs|ERROR"
```
You want to see `Starting server on port: 5044` and `Pipeline started`.

**Confirm the port is listening:**
```bash
ss -lntp | grep 5044
```

> **Password out of the command line (recommended):**
> ```bash
> echo 'ELASTIC_PASSWORD=<your-elastic-password>' > ~/ELK/.env
> chmod 600 ~/ELK/.env
> ```
> then swap `-e ELASTIC_PASSWORD=...` for `--env-file ~/ELK/.env` in the `docker run` above.

---

## Part B — Linux Server Monitoring

### B1. Install Filebeat on the Linux server

On Debian/Ubuntu:

```bash
# Add Elastic's signing key and repository
sudo apt-get install -y apt-transport-https curl gnupg
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch \
  | sudo gpg --dearmor -o /usr/share/keyrings/elastic-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/elastic-keyring.gpg] https://artifacts.elastic.co/packages/9.x/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/elastic-9.x.list

sudo apt-get update
sudo apt-get install -y filebeat
```

On RHEL/CentOS/Rocky:

```bash
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
sudo tee /etc/yum.repos.d/elastic.repo > /dev/null << 'EOF'
[elastic-9.x]
name=Elastic repository for 9.x packages
baseurl=https://artifacts.elastic.co/packages/9.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF

sudo yum install -y filebeat
```

Check the version matches your stack (`9.3.x`):
```bash
filebeat version
```

> **Keep versions aligned.** Filebeat must be the same major version as Elasticsearch, and should not be *newer* than Logstash.

### B2. The `filebeat.yml` for a Linux server

```bash
sudo cp /etc/filebeat/filebeat.yml /etc/filebeat/filebeat.yml.bak
sudo nano /etc/filebeat/filebeat.yml
```

Replace the entire contents with:

```yaml
# ═══════════════════════════════════════════════════════════════════════════
# Filebeat — Linux system log shipper
# File: /etc/filebeat/filebeat.yml
# ═══════════════════════════════════════════════════════════════════════════

filebeat.inputs:

  # ── General system log ───────────────────────────────────────────────────
  - type: filestream
    id: linux-syslog
    enabled: true
    paths:
      - /var/log/syslog          # Debian / Ubuntu
      - /var/log/messages        # RHEL / CentOS / Rocky
    tags: ["linux-system", "syslog"]
    fields:
      log_source: syslog
    fields_under_root: true

  # ── Authentication log (SSH, sudo, su, PAM) ──────────────────────────────
  - type: filestream
    id: linux-auth
    enabled: true
    paths:
      - /var/log/auth.log        # Debian / Ubuntu
      - /var/log/secure          # RHEL / CentOS / Rocky
    tags: ["linux-system", "auth"]
    fields:
      log_source: auth
    fields_under_root: true

  # ── Kernel messages ──────────────────────────────────────────────────────
  - type: filestream
    id: linux-kernel
    enabled: true
    paths:
      - /var/log/kern.log
    tags: ["linux-system", "kernel"]
    fields:
      log_source: kernel
    fields_under_root: true

  # ── Cron jobs (optional — uncomment if your distro logs cron separately) ─
  # - type: filestream
  #   id: linux-cron
  #   enabled: true
  #   paths:
  #     - /var/log/cron.log
  #   tags: ["linux-system", "cron"]

  # ── Multiline example: stack traces in an app log ────────────────────────
  # - type: filestream
  #   id: app-log
  #   enabled: true
  #   paths:
  #     - /var/log/myapp/*.log
  #   tags: ["linux-system", "app"]
  #   parsers:
  #     - multiline:
  #         type: pattern
  #         pattern: '^[[:space:]]|^Caused by:'
  #         negate: false
  #         match: after

# ── systemd-journald (for distros that no longer write /var/log/syslog) ────
# On modern systemd-only systems (e.g. minimal Ubuntu 24.04, RHEL 9),
# enable this instead of the syslog filestream input above.
# filebeat.inputs:
#   - type: journald
#     id: linux-journald
#     enabled: true
#     tags: ["linux-system", "journald"]
#     include_matches:
#       - "_SYSTEMD_UNIT=sshd.service"
#       - "_SYSTEMD_UNIT=cron.service"
#       #  remove include_matches entirely to collect the whole journal

# ═══════════════════════════════════════════════════════════════════════════
# PROCESSORS — enrich every event before it leaves the server
# ═══════════════════════════════════════════════════════════════════════════
processors:
  - add_host_metadata:            # hostname, OS name/version, architecture, IPs
      netinfo.enabled: true
  - add_fields:
      target: ''                  # write at the top level
      fields:
        environment: production   # change per server: dev / staging / production
        datacenter: wsl-lab
  - drop_fields:
      fields: ["agent.ephemeral_id", "ecs.version"]
      ignore_missing: true

  # Example: stop shipping the noisiest repeating line
  # - drop_event:
  #     when:
  #       contains:
  #         message: "CRON[.*]: pam_unix(cron:session)"

# ═══════════════════════════════════════════════════════════════════════════
# OUTPUT — ship to the central Logstash receiver
# ═══════════════════════════════════════════════════════════════════════════
output.logstash:
  # Same machine as Logstash  -> localhost
  # Another server on the LAN -> the Windows host IP with the portproxy rule
  hosts: ["localhost:5044"]
  bulk_max_size: 2048
  # Resilience: keep retrying instead of dropping when Logstash is down
  backoff.init: 1s
  backoff.max: 60s

# Never enable both outputs at once — comment one out.
# output.elasticsearch:
#   hosts: ["https://<wsl-ip>:9200"]
#   username: "elastic"
#   password: "<your-elastic-password>"
#   ssl.certificate_authorities: ["/etc/filebeat/certs/http_ca.crt"]

# ═══════════════════════════════════════════════════════════════════════════
# FILEBEAT'S OWN LOGGING — for debugging the agent itself
# ═══════════════════════════════════════════════════════════════════════════
logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
  permissions: 0640
```

**What each block does:**

| Block | Purpose |
|---|---|
| `type: filestream` | The modern input for tailing files (replaces the deprecated `type: log`). Each input needs a unique `id` |
| `tags: ["linux-system", ...]` | **The routing key.** Logstash branches on `linux-system` to pick the grok filter and the index |
| `fields` + `fields_under_root: true` | Adds your own top-level fields (e.g. `log_source`) instead of nesting them under `fields.*` |
| `add_host_metadata` | Stamps hostname, OS, kernel version, IP addresses onto every event — this is how you tell servers apart in Kibana |
| `add_fields: environment` | Lets you filter `environment: "production"` across every server at once |
| `drop_fields` / `drop_event` | Cut noise and index size before it costs you disk |
| `parsers: multiline` | Joins a Java/Python stack trace into a single event instead of 40 separate ones |
| `backoff.*` | If Logstash is down, Filebeat retries and resumes from its saved offset — no gaps |

### B3. Validate and start

```bash
# Syntax check — do this before every restart
sudo filebeat test config -c /etc/filebeat/filebeat.yml

# Connectivity check — proves Filebeat can reach Logstash on 5044
sudo filebeat test output

# Start and enable at boot
sudo systemctl enable filebeat
sudo systemctl start filebeat
sudo systemctl status filebeat
```

`filebeat test output` should print:
```
logstash: localhost:5044...
  connection...
    parse host... OK
    dns lookup... OK
    dial up... OK
  TLS... WARN secure connection disabled
  talk to server... OK
```

### B4. Generate some test events

```bash
# Write a line into syslog
logger "ELK TEST: hello from $(hostname)"

# Produce an auth success + failure
sudo -k; sudo true                  # sudo authentication event
ssh nosuchuser@localhost            # generates "Invalid user" in auth.log (Ctrl+C to abort)

# Watch Filebeat pick it up
sudo tail -f /var/log/filebeat/filebeat.ndjson
```

### B5. Optional — use the Filebeat `system` module instead

The `system` module bundles ready-made parsing and Kibana dashboards:

```bash
sudo filebeat modules enable system
sudo nano /etc/filebeat/modules.d/system.yml   # set syslog.enabled / auth.enabled to true
```

> **Caveat with a Logstash output:** module parsing runs as an **Elasticsearch ingest pipeline**, not in Filebeat. When output goes to Logstash, you must load those pipelines once (`sudo filebeat setup --pipelines --modules system -E output.logstash.enabled=false -E 'output.elasticsearch.hosts=["https://<wsl-ip>:9200"]' ...`) **and** add `pipeline => "%{[@metadata][pipeline]}"` to the Logstash `elasticsearch` output.
>
> The custom `filestream` + grok approach in this guide avoids that dependency entirely and is easier to reason about. Use the module only if you specifically want the prebuilt dashboards.

---

## Part C — Windows Server Monitoring

Filebeat's `winlog` input reads the Windows Event Log directly — you do **not** need a separate Winlogbeat installation.

### C1. Install Filebeat on the Windows server

In an **elevated PowerShell** (Run as Administrator):

```powershell
# 1. Download and extract
$ver = "9.3.2"
Invoke-WebRequest -Uri "https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-$ver-windows-x86_64.zip" `
  -OutFile "$env:TEMP\filebeat.zip"
Expand-Archive -Path "$env:TEMP\filebeat.zip" -DestinationPath "C:\Program Files" -Force
Rename-Item "C:\Program Files\filebeat-$ver-windows-x86_64" "Filebeat"

# 2. Install as a Windows service
cd "C:\Program Files\Filebeat"
.\install-service-filebeat.ps1
```

If the script is blocked by execution policy:
```powershell
PowerShell.exe -ExecutionPolicy UnRestricted -File .\install-service-filebeat.ps1
```

### C2. The `filebeat.yml` for a Windows server

Edit `C:\Program Files\Filebeat\filebeat.yml`:

```yaml
# ═══════════════════════════════════════════════════════════════════════════
# Filebeat — Windows Server log shipper
# File: C:\Program Files\Filebeat\filebeat.yml
# ═══════════════════════════════════════════════════════════════════════════

filebeat.inputs:

  # ── Security channel: logons, privilege use, account changes ─────────────
  - type: winlog
    name: Security
    tags: ["windows-system", "security"]
    fields:
      log_source: security
    fields_under_root: true
    # Only the event IDs that matter — the Security channel is enormous
    event_id: 4624, 4625, 4634, 4648, 4672, 4720, 4722, 4725, 4726, 4740, 1102
    ignore_older: 72h

  # ── System channel: services, drivers, shutdowns ─────────────────────────
  - type: winlog
    name: System
    tags: ["windows-system", "system"]
    fields:
      log_source: system
    fields_under_root: true
    ignore_older: 72h

  # ── Application channel: app crashes and errors only ─────────────────────
  - type: winlog
    name: Application
    tags: ["windows-system", "application"]
    fields:
      log_source: application
    fields_under_root: true
    # level filtering via XML query — errors and warnings only
    event_id: 1000, 1001, 1002
    ignore_older: 72h

  # ── PowerShell script block logging (optional, security-relevant) ────────
  # - type: winlog
  #   name: Microsoft-Windows-PowerShell/Operational
  #   tags: ["windows-system", "powershell"]
  #   event_id: 4103, 4104

  # ── IIS access logs (optional — file based, not event log) ───────────────
  # - type: filestream
  #   id: iis-access
  #   enabled: true
  #   paths:
  #     - C:\inetpub\logs\LogFiles\*\*.log
  #   tags: ["windows-system", "iis"]
  #   exclude_lines: ['^#']          # skip the W3C header lines

# ═══════════════════════════════════════════════════════════════════════════
# PROCESSORS
# ═══════════════════════════════════════════════════════════════════════════
processors:
  - add_host_metadata:
      netinfo.enabled: true
  - add_fields:
      target: ''
      fields:
        environment: production
        datacenter: on-prem
  # The rendered XML doubles event size and is rarely queried
  - drop_fields:
      fields: ["winlog.xml", "agent.ephemeral_id", "ecs.version"]
      ignore_missing: true

  # Example: ignore the machine-account logons that flood the Security log
  # - drop_event:
  #     when:
  #       regexp:
  #         winlog.event_data.TargetUserName: '.*\$$'

# ═══════════════════════════════════════════════════════════════════════════
# OUTPUT — the same central Logstash on WSL2
# ═══════════════════════════════════════════════════════════════════════════
output.logstash:
  # If Filebeat runs on the SAME Windows machine that hosts WSL2:
  hosts: ["localhost:5044"]
  # If it runs on a DIFFERENT server, use the WSL2 host's LAN IP:
  # hosts: ["192.168.1.50:5044"]
  bulk_max_size: 2048
  backoff.init: 1s
  backoff.max: 60s

# ═══════════════════════════════════════════════════════════════════════════
# FILEBEAT'S OWN LOGGING
# ═══════════════════════════════════════════════════════════════════════════
logging.level: info
logging.to_files: true
logging.files:
  path: C:\ProgramData\filebeat\logs
  name: filebeat
  keepfiles: 7
```

**Windows-specific notes:**

| Setting | Why it matters |
|---|---|
| `type: winlog` | Reads the Event Log API directly — no file paths, no permissions on log files |
| `name: Security` | The channel name exactly as it appears in Event Viewer. For custom channels use the full path, e.g. `Microsoft-Windows-Sysmon/Operational` |
| `event_id:` allow-list | Without it the Security channel alone can push millions of events a day and fill your disk |
| `ignore_older: 72h` | On first start Filebeat would otherwise read the entire retained event log history |
| `drop_fields: winlog.xml` | The raw XML roughly doubles document size for information you already have as fields |
| Service account | Must be `LocalSystem` (the install script's default) to read the **Security** channel |

### C3. Validate and start

```powershell
cd "C:\Program Files\Filebeat"

# Syntax check
.\filebeat.exe test config -c .\filebeat.yml -e

# Connectivity check
.\filebeat.exe test output -c .\filebeat.yml -e

# Start the service
Start-Service filebeat
Get-Service filebeat

# Make it start at boot
Set-Service -Name filebeat -StartupType Automatic
```

### C4. Generate a test event

```powershell
# Write a custom entry to the Application log
New-EventLog -LogName Application -Source "ELKTest" -ErrorAction SilentlyContinue
Write-EventLog -LogName Application -Source "ELKTest" -EventId 1000 `
  -EntryType Information -Message "ELK TEST: hello from $env:COMPUTERNAME"

# Generate a failed logon (4625) — a deliberately wrong password
runas /user:nosuchuser cmd.exe

# Watch Filebeat's own log
Get-Content C:\ProgramData\filebeat\logs\filebeat-*.ndjson -Tail 20 -Wait
```

---

## Part D — Verify the Data Reached Elasticsearch

Back on WSL2:

```bash
curl --cacert ~/ELK/http_ca.crt \
  -u "elastic:<your-elastic-password>" \
  "https://localhost:9200/_cat/indices/*-system-*?v"
```

Expected:
```
health status index                      docs.count store.size
yellow open   linux-system-2026.08.09           4213      2.1mb
yellow open   windows-system-2026.08.09          876      1.4mb
```

**Look at one document to confirm the parsing worked:**
```bash
curl --cacert ~/ELK/http_ca.crt \
  -u "elastic:<your-elastic-password>" \
  "https://localhost:9200/linux-system-*/_search?size=1&pretty"
```

You should see `process`, `log_message`, `server_name`, `os_family` as real fields — not everything crammed into `message`.

**Check for parse failures:**
```bash
curl --cacert ~/ELK/http_ca.crt \
  -u "elastic:<your-elastic-password>" \
  "https://localhost:9200/linux-system-*/_count?q=tags:_syslog_grok_failure&pretty"
```
A small count is normal (multi-line kernel dumps). A large count means your distro's syslog format differs — adjust the grok pattern.

---

## Part E — View It in Kibana

### E1. Create the Data Views

1. Open `http://localhost:5601`, log in as `elastic`
2. Menu → **Stack Management** → **Data Views** → **Create data view**

| Name | Index pattern | Timestamp field |
|---|---|---|
| `Linux System Logs` | `linux-system-*` | `@timestamp` |
| `Windows System Logs` | `windows-system-*` | `@timestamp` |
| `All Server Logs` | `*-system-*` | `@timestamp` |

### E2. Useful columns in Discover

**Linux:** `server_name`, `process`, `log_message`, `event_outcome`, `source_ip`
**Windows:** `computer_name`, `event_id`, `event_name`, `event_provider`, `log_message`

### E3. KQL queries worth saving

**Linux — security**
```
# Failed SSH logins
tags: "auth_failure" and os_family: "linux"

# All sudo usage
process: "sudo"

# SSH activity from a specific IP
process: "sshd" and source_ip: "10.0.0.5"

# Everything except the noisy cron chatter
NOT process: "CRON"

# Service failures reported by systemd
process: "systemd" and log_message: *Failed*

# Out-of-memory killer fired
log_source: "kernel" and log_message: *"Out of memory"*
```

**Windows — security and health**
```
# Failed logons
event_id: 4625

# Successful logons for a specific user
event_id: 4624 and winlog.event_data.TargetUserName: "administrator"

# New user accounts created
event_id: 4720

# Account lockouts
event_id: 4740

# Audit log was cleared (a classic red flag)
event_id: 1102

# Unexpected shutdowns / restarts
event_id: (1074 or 6006 or 6008)

# Service state changes
event_id: 7036 and event_channel: "System"
```

**Across both**
```
# Everything from one host
server_name: "web01"

# Only production servers
environment: "production"

# Any authentication failure, any OS
event_category: "authentication" and event_outcome: "failure"
```

### E4. A starter dashboard

1. **Dashboard** → **Create dashboard** → **Create visualization** (Lens)
2. Build these four panels:

| Panel | Chart type | Configuration |
|---|---|---|
| Events over time by server | Bar vertical stacked | X-axis `@timestamp`, Break down by `server_name` |
| Failed logins | Metric | Count, filtered to `tags: "auth_failure"` |
| Top Windows event IDs | Horizontal bar | Y-axis top 10 values of `event_id` |
| Busiest Linux processes | Pie | Slice by `process`, top 10 |

3. **Save** as `Server Health Overview`

### E5. Optional — alert on failed logins

**Stack Management → Rules → Create rule → Elasticsearch query**
- Index: `*-system-*`
- Query: `tags: "auth_failure"`
- Condition: count **is above 10** in the last **5 minutes**
- Action: Server log / email / webhook connector

---

## Managing Index Growth

System logs grow fast. Set retention before your disk fills:

```bash
# Delete indices older than a chosen date, by hand
curl --cacert ~/ELK/http_ca.crt -u "elastic:<your-elastic-password>" \
  -X DELETE "https://localhost:9200/linux-system-2026.07.*"

# Check how much space each index is using
curl --cacert ~/ELK/http_ca.crt -u "elastic:<your-elastic-password>" \
  "https://localhost:9200/_cat/indices/*-system-*?v&s=store.size:desc&h=index,docs.count,store.size"
```

**Automate with an ILM policy (7-day retention):**
```bash
curl --cacert ~/ELK/http_ca.crt -u "elastic:<your-elastic-password>" \
  -X PUT "https://localhost:9200/_ilm/policy/system-logs-policy" \
  -H 'Content-Type: application/json' -d '{
  "policy": {
    "phases": {
      "hot":    { "actions": {} },
      "delete": { "min_age": "7d", "actions": { "delete": {} } }
    }
  }
}'

# Apply it to all future system indices via an index template
curl --cacert ~/ELK/http_ca.crt -u "elastic:<your-elastic-password>" \
  -X PUT "https://localhost:9200/_index_template/system-logs-template" \
  -H 'Content-Type: application/json' -d '{
  "index_patterns": ["linux-system-*", "windows-system-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0,
      "index.lifecycle.name": "system-logs-policy"
    }
  }
}'
```

> `number_of_replicas: 0` is what turns the index health from `yellow` to `green` on a single-node cluster.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `filebeat test output` → `dial up... ERROR` | Logstash not running, or port not reachable | `docker ps \| grep logstash-sys`; check `portproxy` and firewall |
| Filebeat running, no index appears | Events reach Logstash but the output errors out | `docker logs logstash-sys --tail 50` — look for `401` or SSL errors |
| Logstash logs `401 Unauthorized` | `ELASTIC_PASSWORD` wrong in the container | Recreate the container with the correct password (see below) |
| Logstash logs `Connection refused to es01:9200` | Logstash is not on the `elastic` network | `docker network connect elastic logstash-sys` |
| Everything lands in `unclassified-*` | The `tags:` in `filebeat.yml` don't match the Logstash `if` conditions | Ensure `linux-system` / `windows-system` are spelled exactly |
| Linux events show wrong time in Kibana | `timezone` in the `date` filter doesn't match the server | Set the correct zone in `logstash.conf` and restart Logstash |
| `tags: _syslog_grok_failure` on most events | Distro uses a different syslog format (e.g. RFC5424) | Add a second `grok` match pattern, or switch to the `journald` input |
| Windows Security channel empty | Filebeat service isn't `LocalSystem`, or auditing is off | Check the service logon account; enable audit policy via `gpedit.msc` |
| Windows: `The specified channel could not be found` | Channel name typo | Copy the exact name from Event Viewer → *Properties → Full Name* |
| Filebeat re-sends everything after reinstall | Registry (offset store) was deleted | Normal — Linux: `/var/lib/filebeat/registry`, Windows: `C:\ProgramData\filebeat\registry` |
| Disk filling rapidly | Too many event IDs / no retention | Tighten the `event_id` allow-list; apply the ILM policy above |

**Restart Logstash after a config change:**
```bash
docker restart logstash-sys
docker logs -f logstash-sys
```

**Recreate Logstash with a new password:**
```bash
docker rm -f logstash-sys
# re-run the docker run command from Part A2 with the correct password
```

**Turn on Filebeat debug output when nothing else explains it:**
```bash
# Linux — run in the foreground with debug logging
sudo systemctl stop filebeat
sudo filebeat -e -d "publish,logstash"
```
```powershell
# Windows
Stop-Service filebeat
cd "C:\Program Files\Filebeat"; .\filebeat.exe -e -d "publish,logstash"
```

---

## Optional: Adding Metrics with Metricbeat

Filebeat covers logs. For CPU, memory, disk, and network **metrics**, add Metricbeat on the same servers — it can ship straight to Elasticsearch (no Logstash needed):

```bash
sudo apt-get install -y metricbeat
sudo metricbeat modules enable system

# Point it at Elasticsearch (needs the CA cert copied to the server)
sudo nano /etc/metricbeat/metricbeat.yml
```
```yaml
output.elasticsearch:
  hosts: ["https://<wsl-ip>:9200"]
  username: "elastic"
  password: "<your-elastic-password>"
  ssl.certificate_authorities: ["/etc/metricbeat/certs/http_ca.crt"]
  ssl.verification_mode: "none"

setup.kibana:
  host: "http://<wsl-ip>:5601"
```
```bash
sudo metricbeat setup          # loads the prebuilt Kibana dashboards
sudo systemctl enable --now metricbeat
```

Then open **Kibana → Dashboards → [Metricbeat System] Host overview** for ready-made CPU/memory/disk charts.

---

## Quick Reference

| Task | Command |
|---|---|
| Start the Logstash receiver | `docker start logstash-sys` |
| Watch Logstash logs | `docker logs -f logstash-sys` |
| Filebeat status (Linux) | `sudo systemctl status filebeat` |
| Filebeat status (Windows) | `Get-Service filebeat` |
| Validate Filebeat config | `sudo filebeat test config` |
| Test the Logstash connection | `sudo filebeat test output` |
| Restart Filebeat (Linux) | `sudo systemctl restart filebeat` |
| Restart Filebeat (Windows) | `Restart-Service filebeat` |
| List system indices | `curl --cacert ~/ELK/http_ca.crt -u elastic:<pass> "https://localhost:9200/_cat/indices/*-system-*?v"` |
| Refresh WSL2 port forward | `netsh interface portproxy add v4tov4 listenport=5044 connectaddress=$(wsl hostname -I) connectport=5044` |

| Component | Where it runs | Port | Config file |
|---|---|---|---|
| Elasticsearch (`es01`) | WSL2 Docker | 9200 | — |
| Kibana (`kib01`) | WSL2 Docker | 5601 | — |
| Logstash (`logstash-sys`) | WSL2 Docker | 5044 | `~/ELK/logstash-sys/pipeline/logstash.conf` |
| Filebeat (Linux) | On each Linux server | — | `/etc/filebeat/filebeat.yml` |
| Filebeat (Windows) | On each Windows server | — | `C:\Program Files\Filebeat\filebeat.yml` |

---

*For ELK Stack installation, see [elk-single-node-docker-wsl2-setup.md](./elk-single-node-docker-wsl2-setup.md)*
*For application log shipping, see [elk-app-log-integration.md](./elk-app-log-integration.md)*
*For query examples, see [elk-kibana-sample-queries.md](./elk-kibana-sample-queries.md)*
*For complete teardown, see [elk-stack-complete-teardown.md](./elk-stack-complete-teardown.md)*

*Version: 9.3.2 | Platform: WSL2 + Docker Desktop | Single Node Setup*
