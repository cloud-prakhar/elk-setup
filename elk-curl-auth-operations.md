# ELK Stack — curl Operations with Password and API Key Auth

This guide shows how to run Elasticsearch operations from the command line with **curl**, using **two authentication methods side by side**:

1. **Basic auth** — username + password (the `elastic` user)
2. **API key** — a token you create once and use in an `Authorization: ApiKey` header

Every command below has been run against a live single-node ELK stack (Elasticsearch 9.3.2 over HTTPS). For each operation you get the password version first, then the equivalent API-key version, so you can pick whichever fits your situation.

**Applies to:** the single-node stack from `elk-single-node-docker-wsl2-setup.md` (Elasticsearch on `https://localhost:9200`, security enabled, self-signed CA at `~/ELK/http_ca.crt`).

---

## Why two methods? Password vs API key

| | Password (Basic auth) | API key |
|---|---|---|
| What it is | The `elastic` superuser login | A scoped token (id + secret) you generate |
| Best for | Quick admin/debugging from your own machine | Apps, scripts, CI, cron — anything automated |
| Privileges | Full superuser (can do anything) | Exactly the privileges you grant it |
| Revoking | Must change the password (affects everything) | Invalidate just that one key |
| Expiry | Never (until rotated) | Optional `expiration` (e.g. `1d`, `30d`) |
| Risk if leaked | Total cluster compromise | Limited to the key's granted scope |

**Rule of thumb:** use the password to set things up and to create API keys. Use API keys for everything an app or script does. Never hard-code the `elastic` password into application code.

---

## Setup: shared variables

Run these once per terminal session so the rest of the commands stay short. Replace the password with the one you saved in **Step 5a** of the setup guide.

```bash
# Connection details
export ES="https://localhost:9200"
export CA="$HOME/ELK/http_ca.crt"

# --- Method 1: password (basic auth) ---
export ES_USER="elastic"
export ES_PASS="<your-elastic-password>"

# --- Method 2: API key (created in the next section) ---
export ES_APIKEY="<paste-the-encoded-api-key-here>"
```

> The `--cacert $CA` flag tells curl to trust the stack's self-signed certificate. Without it you get a TLS error. (For throwaway local testing only, `-k` skips verification — never use `-k` against anything real.)

A quick connectivity check with each method:

```bash
# Password
curl --cacert "$CA" -u "$ES_USER:$ES_PASS" "$ES"

# API key (after you create one below)
curl --cacert "$CA" -H "Authorization: ApiKey $ES_APIKEY" "$ES"
```

Both should return the cluster's "You Know, for Search" banner JSON.

---

## Part 1 — Create and manage API keys (password required)

You create an API key **using the password** (or another key with the right privileges). The response includes an `encoded` value — that is what goes into the `Authorization: ApiKey` header.

### Create a scoped API key

```bash
curl --cacert "$CA" -u "$ES_USER:$ES_PASS" \
  -H 'Content-Type: application/json' \
  -XPOST "$ES/_security/api_key" -d '
{
  "name": "my-app-key",
  "expiration": "30d",
  "role_descriptors": {
    "logs_rw": {
      "cluster": ["monitor"],
      "indices": [
        {
          "names": ["sample-*", "flask-app-*"],
          "privileges": ["create_index", "read", "write", "view_index_metadata", "manage"]
        }
      ]
    }
  }
}'
```

Response looks like:

```json
{
  "id": "TaGroJ4BH1KaEdOtmIdI",
  "name": "my-app-key",
  "expiration": 1780898460749,
  "api_key": "mfawxx24EUzuGC10_SvyRA",
  "encoded": "VGFHcm9KNEJIMUthRWRPdG1JZEk6bWZhd3h4MjRFVXp1R0MxMF9TdnlSQQ=="
}
```

Take the **`encoded`** field and store it:

```bash
export ES_APIKEY="VGFHcm9KNEJIMUthRWRPdG1JZEk6bWZhd3h4MjRFVXp1R0MxMF9TdnlSQQ=="
```

> - `encoded` = base64 of `id:api_key`, ready to drop into the header. This is the easy path.
> - The raw `api_key` secret is shown **only once** — if you lose it, invalidate the key and make a new one.
> - Omit `role_descriptors` to create a key with the **same privileges as the creating user** (the `elastic` superuser) — convenient but broad; prefer scoping it as shown.

### List and inspect keys (password)

```bash
# All keys you own
curl --cacert "$CA" -u "$ES_USER:$ES_PASS" "$ES/_security/api_key?pretty"

# Who am I / what can this API key do? (run WITH the key)
curl --cacert "$CA" -H "Authorization: ApiKey $ES_APIKEY" "$ES/_security/_authenticate?pretty"
```

### Invalidate (revoke) a key (password)

```bash
# By name
curl --cacert "$CA" -u "$ES_USER:$ES_PASS" \
  -H 'Content-Type: application/json' \
  -XDELETE "$ES/_security/api_key" -d '{"name":"my-app-key"}'

# By id
curl --cacert "$CA" -u "$ES_USER:$ES_PASS" \
  -H 'Content-Type: application/json' \
  -XDELETE "$ES/_security/api_key" -d '{"ids":["TaGroJ4BH1KaEdOtmIdI"]}'
```

---

## Part 2 — Operations, each shown with BOTH methods

The only thing that changes between the two methods is the auth flag:

```
Password : -u "$ES_USER:$ES_PASS"
API key  : -H "Authorization: ApiKey $ES_APIKEY"
```

Everything after that (method, path, body) is identical. Below, each operation lists the password command first, then the API-key command.

### 2.1 Cluster / index info — GET

```bash
# List all indices (password)
curl --cacert "$CA" -u "$ES_USER:$ES_PASS" "$ES/_cat/indices?v"
# List all indices (API key)
curl --cacert "$CA" -H "Authorization: ApiKey $ES_APIKEY" "$ES/_cat/indices?v"

# Cluster health (password)
curl --cacert "$CA" -u "$ES_USER:$ES_PASS" "$ES/_cluster/health?pretty"
# Cluster health (API key)
curl --cacert "$CA" -H "Authorization: ApiKey $ES_APIKEY" "$ES/_cluster/health?pretty"

# Count docs in an index pattern (password)
curl --cacert "$CA" -u "$ES_USER:$ES_PASS" "$ES/flask-app-*/_count?pretty"
# (API key)
curl --cacert "$CA" -H "Authorization: ApiKey $ES_APIKEY" "$ES/flask-app-*/_count?pretty"
```

### 2.2 Create an index — PUT

```bash
# Password
curl --cacert "$CA" -u "$ES_USER:$ES_PASS" \
  -H 'Content-Type: application/json' \
  -XPUT "$ES/sample-index" -d '
{
  "settings": {"number_of_shards": 1, "number_of_replicas": 0},
  "mappings": {
    "properties": {
      "name":       {"type": "text"},
      "level":      {"type": "keyword"},
      "age":        {"type": "integer"},
      "created_at": {"type": "date"}
    }
  }
}'

# API key
curl --cacert "$CA" -H "Authorization: ApiKey $ES_APIKEY" \
  -H 'Content-Type: application/json' \
  -XPUT "$ES/sample-index" -d '
{
  "settings": {"number_of_shards": 1, "number_of_replicas": 0},
  "mappings": {
    "properties": {
      "name":       {"type": "text"},
      "level":      {"type": "keyword"},
      "age":        {"type": "integer"},
      "created_at": {"type": "date"}
    }
  }
}'
```

### 2.3 Add data — POST (auto id) and PUT (known id)

```bash
# POST = auto-generated id (password)
curl --cacert "$CA" -u "$ES_USER:$ES_PASS" \
  -H 'Content-Type: application/json' \
  -XPOST "$ES/sample-index/_doc" -d '{"name":"Alice","level":"INFO","age":30,"created_at":"2026-06-07T10:00:00"}'
# POST (API key)
curl --cacert "$CA" -H "Authorization: ApiKey $ES_APIKEY" \
  -H 'Content-Type: application/json' \
  -XPOST "$ES/sample-index/_doc" -d '{"name":"Alice","level":"INFO","age":30,"created_at":"2026-06-07T10:00:00"}'

# PUT = create/replace at a known id (password)
curl --cacert "$CA" -u "$ES_USER:$ES_PASS" \
  -H 'Content-Type: application/json' \
  -XPUT "$ES/sample-index/_doc/1" -d '{"name":"Bob","level":"ERROR","age":45}'
# PUT (API key)
curl --cacert "$CA" -H "Authorization: ApiKey $ES_APIKEY" \
  -H 'Content-Type: application/json' \
  -XPUT "$ES/sample-index/_doc/1" -d '{"name":"Bob","level":"ERROR","age":45}'
```

### 2.4 Bulk insert — POST `_bulk`

`_bulk` needs newline-delimited JSON. The cleanest way in a shell is a heredoc with `--data-binary` so curl preserves the newlines.

```bash
# Password
curl --cacert "$CA" -u "$ES_USER:$ES_PASS" \
  -H 'Content-Type: application/x-ndjson' \
  -XPOST "$ES/_bulk" --data-binary '
{"index":{"_index":"sample-index","_id":"3"}}
{"name":"Dave","level":"INFO","age":52}
{"index":{"_index":"sample-index","_id":"4"}}
{"name":"Eve","level":"ERROR","age":19}
'

# API key
curl --cacert "$CA" -H "Authorization: ApiKey $ES_APIKEY" \
  -H 'Content-Type: application/x-ndjson' \
  -XPOST "$ES/_bulk" --data-binary '
{"index":{"_index":"sample-index","_id":"3"}}
{"name":"Dave","level":"INFO","age":52}
{"index":{"_index":"sample-index","_id":"4"}}
{"name":"Eve","level":"ERROR","age":19}
'
```

> Note `Content-Type: application/x-ndjson` and `--data-binary` (not `-d`) for bulk — `-d` strips newlines and the request fails.

### 2.5 Retrieve data — GET

```bash
# Get one doc by id (password)
curl --cacert "$CA" -u "$ES_USER:$ES_PASS" "$ES/sample-index/_doc/1?pretty"
# (API key)
curl --cacert "$CA" -H "Authorization: ApiKey $ES_APIKEY" "$ES/sample-index/_doc/1?pretty"

# Search (password)
curl --cacert "$CA" -u "$ES_USER:$ES_PASS" \
  -H 'Content-Type: application/json' \
  -XGET "$ES/sample-index/_search?pretty" -d '{"query":{"match":{"level":"ERROR"}}}'
# Search (API key)
curl --cacert "$CA" -H "Authorization: ApiKey $ES_APIKEY" \
  -H 'Content-Type: application/json' \
  -XGET "$ES/sample-index/_search?pretty" -d '{"query":{"match":{"level":"ERROR"}}}'
```

### 2.6 Update / patch data — POST `_update`

Elasticsearch has **no HTTP `PATCH`** — partial updates use `POST .../_update/<id>`.

```bash
# Partial update (password)
curl --cacert "$CA" -u "$ES_USER:$ES_PASS" \
  -H 'Content-Type: application/json' \
  -XPOST "$ES/sample-index/_update/1" -d '{"doc":{"age":46,"level":"CRITICAL"}}'
# (API key)
curl --cacert "$CA" -H "Authorization: ApiKey $ES_APIKEY" \
  -H 'Content-Type: application/json' \
  -XPOST "$ES/sample-index/_update/1" -d '{"doc":{"age":46,"level":"CRITICAL"}}'

# Scripted update (password)
curl --cacert "$CA" -u "$ES_USER:$ES_PASS" \
  -H 'Content-Type: application/json' \
  -XPOST "$ES/sample-index/_update/1" -d '{"script":{"source":"ctx._source.age += params.n","params":{"n":1}}}'
# (API key)
curl --cacert "$CA" -H "Authorization: ApiKey $ES_APIKEY" \
  -H 'Content-Type: application/json' \
  -XPOST "$ES/sample-index/_update/1" -d '{"script":{"source":"ctx._source.age += params.n","params":{"n":1}}}'
```

### 2.7 Delete data — DELETE

```bash
# Delete one doc (password)
curl --cacert "$CA" -u "$ES_USER:$ES_PASS" -XDELETE "$ES/sample-index/_doc/4"
# (API key)
curl --cacert "$CA" -H "Authorization: ApiKey $ES_APIKEY" -XDELETE "$ES/sample-index/_doc/4"

# Delete by query (password)
curl --cacert "$CA" -u "$ES_USER:$ES_PASS" \
  -H 'Content-Type: application/json' \
  -XPOST "$ES/sample-index/_delete_by_query" -d '{"query":{"range":{"age":{"lt":20}}}}'
# (API key)
curl --cacert "$CA" -H "Authorization: ApiKey $ES_APIKEY" \
  -H 'Content-Type: application/json' \
  -XPOST "$ES/sample-index/_delete_by_query" -d '{"query":{"range":{"age":{"lt":20}}}}'

# Delete the whole index — IRREVERSIBLE (password)
curl --cacert "$CA" -u "$ES_USER:$ES_PASS" -XDELETE "$ES/sample-index"
# (API key)
curl --cacert "$CA" -H "Authorization: ApiKey $ES_APIKEY" -XDELETE "$ES/sample-index"
```

> **Warning:** `DELETE /index` and `_delete_by_query` cannot be undone. Don't run them against `flask-app-*` (your real logs) unless you mean to wipe them, and never run `DELETE /*`.

---

## Part 3 — End-to-end smoke test (copy/paste)

This runs the full lifecycle with the **API key** and cleans up after itself. (Swap the auth flag to test the password path.)

```bash
AUTH=(-H "Authorization: ApiKey $ES_APIKEY")   # or: AUTH=(-u "$ES_USER:$ES_PASS")
JSON=(-H 'Content-Type: application/json')

echo "1. create index";  curl --cacert "$CA" "${AUTH[@]}" -XPUT "$ES/sample-index" "${JSON[@]}" -d '{"settings":{"number_of_replicas":0}}'; echo
echo "2. add doc id=1";   curl --cacert "$CA" "${AUTH[@]}" -XPUT "$ES/sample-index/_doc/1" "${JSON[@]}" -d '{"name":"Bob","level":"ERROR","age":45}'; echo
echo "3. refresh";        curl --cacert "$CA" "${AUTH[@]}" -XPOST "$ES/sample-index/_refresh"; echo
echo "4. read doc";       curl --cacert "$CA" "${AUTH[@]}" "$ES/sample-index/_doc/1?pretty"
echo "5. patch doc";      curl --cacert "$CA" "${AUTH[@]}" -XPOST "$ES/sample-index/_update/1" "${JSON[@]}" -d '{"doc":{"level":"INFO"}}'; echo
echo "6. search";         curl --cacert "$CA" "${AUTH[@]}" "$ES/sample-index/_search?pretty" "${JSON[@]}" -d '{"query":{"match_all":{}}}'
echo "7. delete doc";     curl --cacert "$CA" "${AUTH[@]}" -XDELETE "$ES/sample-index/_doc/1"; echo
echo "8. delete index";   curl --cacert "$CA" "${AUTH[@]}" -XDELETE "$ES/sample-index"; echo
```

---

## HTTP verb quick reference

| Verb | Operation | Example path |
|---|---|---|
| `GET` | Read doc / search / metadata | `/index/_doc/1`, `/index/_search`, `/_cat/indices` |
| `POST` | Create (auto-id), `_update`, `_bulk`, `_delete_by_query` | `/index/_doc`, `/index/_update/1` |
| `PUT` | Create index, create/replace doc at known id | `/index`, `/index/_doc/1` |
| `PATCH` | **Not supported** — use `POST /index/_update/<id>` | — |
| `DELETE` | Delete doc or index, invalidate API key | `/index/_doc/1`, `/index`, `/_security/api_key` |

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `curl: (60) SSL certificate problem` | CA cert not trusted | Add `--cacert ~/ELK/http_ca.crt` (or re-copy it per setup Step 4) |
| `401 ... missing authentication credentials` | No/with wrong auth | Add `-u elastic:<pass>` or `-H "Authorization: ApiKey <encoded>"` |
| `401 ... unable to authenticate` (API key) | Key invalidated/expired, or used the wrong field | Use the `encoded` value, not the raw `api_key`; create a new key |
| `403 ... action [...] is unauthorized` | API key lacks the privilege | Recreate the key with the needed `role_descriptors` |
| `406 ... Content-Type header ... not supported` | Body sent without header | Add `-H 'Content-Type: application/json'` |
| `_bulk` returns `400` / parse errors | Newlines stripped by `-d` | Use `--data-binary` + `Content-Type: application/x-ndjson`, with a trailing newline |

---

## Security notes

- **Never commit** the `elastic` password or a raw API key. Put them in a `.env` file that is git-ignored (see `elk-app-log-integration.md`).
- Give each app its **own** API key, scoped to only the indices and privileges it needs, with an `expiration`.
- Rotate keys by creating a new one, switching the app over, then invalidating the old one — no downtime, and the `elastic` password never changes.
- In shell history, prefer environment variables (as above) over typing secrets inline so they don't linger in `~/.bash_history`.
