# ELK Stack — Sample Queries and Searches

This document provides ready-to-use queries for exploring logs in Kibana and Elasticsearch. All queries are generic and work across any application sending structured JSON logs to your ELK stack.

**Applies to:** Any app using this elk-setup (demo-app, notes-app, or your own applications)

---

## Quick Reference: Where to Use Each Query Type

| Query Type | Where to use | Best for |
|---|---|---|
| KQL | Discover search bar | Quick filtering, pattern matching |
| Lucene | Discover (switch mode) | Complex text search, wildcards |
| Dev Tools (DSL) | Dev Tools console | Aggregations, analytics, bulk ops |

---

## Part 1 — KQL Queries (Kibana Query Language)

KQL is the default search language in Kibana Discover. Type in the search bar at the top.

### Filter by Log Level

```kql
# Show only errors
level: "ERROR"

# Show warnings and errors
level: ("ERROR" or "WARNING")

# Exclude debug logs
not level: "DEBUG"

# Everything above INFO
level: ("WARNING" or "ERROR" or "CRITICAL")
```

### Filter by Time Range

Use the time picker (top-right) for UI-based time filtering. KQL itself doesn't handle time — use the picker in combination with KQL filters.

### Filter by Endpoint / URL

```kql
# Specific endpoint
endpoint: "/health"

# All note-related endpoints
endpoint: "/notes*"

# Exclude health checks
not endpoint: "/health"
```

### Filter by HTTP Status Code

```kql
# All errors (client + server)
status_code >= 400

# Server errors only
status_code >= 500

# Rate limit responses
status_code: 429

# Successful creates
status_code: 201

# All 2xx responses
status_code >= 200 and status_code < 300
```

### Trace a Single Request

```kql
# Find all log lines for one request (replace with actual request_id)
request_id: "a1b2c3d4"
```

All log lines for a single HTTP request share the same `request_id`. Use this to follow the full lifecycle: request received → processing → response sent.

### Find Slow Requests

```kql
# Requests taking more than 100ms
duration_ms > 100

# Requests taking more than 1 second
duration_ms > 1000
```

### Filter by Application (when multiple apps share an ES cluster)

```kql
app_name: "notes-app"
app_name: "flask-app"
```

### Search Message Text

```kql
# Exact phrase in message
message: "note created"

# Contains any of these words
message: "error" or message: "failed"

# Wildcard (use sparingly — can be slow)
message: "rate*"
```

### Combine Conditions

```kql
# Errors on a specific endpoint
level: "ERROR" and endpoint: "/notes"

# Slow requests that succeeded
duration_ms > 200 and status_code: 200

# Rate-limited requests from a specific IP hash
status_code: 429 and ip_hash: "3f4a9b2e1c8d7f6a"

# Any problem (error or slow)
level: "ERROR" or duration_ms > 500
```

---

## Part 2 — Dev Tools Queries (Elasticsearch Query DSL)

Go to: Hamburger menu → **Dev Tools**

### Index Management

```json
// List all indices
GET /_cat/indices?v

// List only app indices
GET /_cat/indices/notes-app-*?v
GET /_cat/indices/flask-app-*?v

// Count documents in an index pattern
GET /notes-app-*/_count
GET /flask-app-*/_count

// See index mapping (all fields and their types)
GET /notes-app-*/_mapping

// Delete an index (irreversible)
DELETE /notes-app-2026.03.28

// Delete all of an app's indices (irreversible)
DELETE /notes-app-*
```

### Basic Search

```json
// Get the 10 most recent documents
GET /notes-app-*/_search
{
  "size": 10,
  "sort": [{"@timestamp": {"order": "desc"}}]
}

// Find documents by log level
GET /notes-app-*/_search
{
  "query": {
    "term": {"level.keyword": "ERROR"}
  },
  "size": 20,
  "sort": [{"@timestamp": {"order": "desc"}}]
}

// Full-text search in message field
GET /notes-app-*/_search
{
  "query": {
    "match": {"message": "note created"}
  }
}

// Multiple conditions (must = AND)
GET /notes-app-*/_search
{
  "query": {
    "bool": {
      "must": [
        {"term": {"level.keyword": "ERROR"}},
        {"term": {"endpoint.keyword": "/notes"}}
      ]
    }
  }
}

// Exclude a value (must_not = NOT)
GET /notes-app-*/_search
{
  "query": {
    "bool": {
      "must_not": [
        {"term": {"endpoint.keyword": "/health"}}
      ]
    }
  }
}
```

### Range Queries

```json
// Slow requests (duration > 100ms)
GET /notes-app-*/_search
{
  "query": {
    "range": {"duration_ms": {"gt": 100}}
  }
}

// HTTP errors (status_code >= 400)
GET /notes-app-*/_search
{
  "query": {
    "range": {"status_code": {"gte": 400}}
  }
}

// Logs in a specific time range
GET /notes-app-*/_search
{
  "query": {
    "range": {
      "@timestamp": {
        "gte": "now-1h",
        "lte": "now"
      }
    }
  }
}
```

### Aggregations (Analytics)

```json
// Count by log level
GET /notes-app-*/_search
{
  "size": 0,
  "aggs": {
    "by_level": {
      "terms": {"field": "level.keyword"}
    }
  }
}

// Count by endpoint
GET /notes-app-*/_search
{
  "size": 0,
  "aggs": {
    "by_endpoint": {
      "terms": {"field": "endpoint.keyword", "size": 10}
    }
  }
}

// Average, min, max response time
GET /notes-app-*/_search
{
  "size": 0,
  "aggs": {
    "avg_duration": {"avg": {"field": "duration_ms"}},
    "max_duration": {"max": {"field": "duration_ms"}},
    "min_duration": {"min": {"field": "duration_ms"}}
  }
}

// Average response time per endpoint
GET /notes-app-*/_search
{
  "size": 0,
  "aggs": {
    "by_endpoint": {
      "terms": {"field": "endpoint.keyword"},
      "aggs": {
        "avg_duration": {"avg": {"field": "duration_ms"}}
      }
    }
  }
}

// Log count per hour (date histogram)
GET /notes-app-*/_search
{
  "size": 0,
  "aggs": {
    "logs_over_time": {
      "date_histogram": {
        "field": "@timestamp",
        "calendar_interval": "hour"
      }
    }
  }
}

// Error rate: errors per 5-minute bucket
GET /notes-app-*/_search
{
  "size": 0,
  "query": {
    "term": {"level.keyword": "ERROR"}
  },
  "aggs": {
    "errors_over_time": {
      "date_histogram": {
        "field": "@timestamp",
        "fixed_interval": "5m"
      }
    }
  }
}
```

### Request Tracing

```json
// Get all log lines for a specific request_id
GET /notes-app-*/_search
{
  "query": {
    "term": {"request_id.keyword": "a1b2c3d4"}
  },
  "sort": [{"@timestamp": {"order": "asc"}}]
}
```

### Health and Cluster Checks

```json
// Cluster health
GET /_cluster/health

// Node info
GET /_cat/nodes?v

// Index sizes
GET /_cat/indices?v&s=store.size:desc

// Shard allocation
GET /_cat/shards?v
```

---

## Part 3 — Discover Saved Searches (KQL + Columns)

Create these saved searches in Kibana Discover for quick access:

### "All Errors"
- **Query:** `level: "ERROR"`
- **Columns:** `@timestamp`, `message`, `endpoint`, `request_id`, `duration_ms`
- Save as: `All Errors`

### "Slow Requests"
- **Query:** `duration_ms > 200`
- **Columns:** `@timestamp`, `endpoint`, `duration_ms`, `status_code`, `request_id`
- Save as: `Slow Requests`

### "Rate Limited"
- **Query:** `status_code: 429`
- **Columns:** `@timestamp`, `endpoint`, `ip_hash`, `message`
- Save as: `Rate Limited`

### "Health Check Noise Filtered"
- **Query:** `not endpoint: "/health"`
- **Columns:** `@timestamp`, `level`, `message`, `endpoint`, `status_code`
- Save as: `App Logs (No Health)`

---

## Part 4 — Dashboard KQL Filters

Apply these in dashboards using the **Add filter** button (below the search bar) for panel-level filtering:

| Filter | Field | Operator | Value |
|---|---|---|---|
| Errors only | `level` | `is` | `ERROR` |
| Exclude health checks | `endpoint` | `is not` | `/health` |
| Slow requests | `duration_ms` | `is greater than` | `100` |
| Specific app | `app_name` | `is` | `notes-app` |

---

## Part 5 — Common Troubleshooting Queries

### "Why is the index empty?"
```json
// Check when the last document was indexed
GET /notes-app-*/_search
{
  "size": 1,
  "sort": [{"@timestamp": {"order": "desc"}}],
  "_source": ["@timestamp", "message"]
}
```

### "Is Logstash writing to the correct index?"
```json
// List all indices and their document counts
GET /_cat/indices?v&s=index
```

### "What fields does my index actually have?"
```json
// Get the field mapping for the notes-app index
GET /notes-app-*/_mapping

// Get unique values for a field (useful for debugging)
GET /notes-app-*/_search
{
  "size": 0,
  "aggs": {
    "unique_levels": {
      "terms": {"field": "level.keyword"}
    }
  }
}
```

### "Is my @timestamp correct?"
```json
// Compare @timestamp with a known event time
GET /notes-app-*/_search
{
  "size": 5,
  "sort": [{"@timestamp": {"order": "desc"}}],
  "_source": ["@timestamp", "message", "level"]
}
```

If `@timestamp` values look wrong (e.g., year 1970), the date filter in Logstash failed to parse the timestamp — check `logstash.conf` date filter settings and the log format.

---

## Quick Reference Card

```
KIBANA KQL                          DEV TOOLS
─────────────────────────────       ──────────────────────────────────
level: "ERROR"                      GET /index-*/_count
level: ("ERROR" or "WARNING")       GET /index-*/_mapping
not level: "DEBUG"                  GET /_cat/indices?v
status_code >= 400                  DELETE /index-2026.03.28
duration_ms > 100                   GET /_cluster/health
request_id: "abc12345"
endpoint: "/notes"
app_name: "notes-app"
```
