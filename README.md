# Mock Server

HTTP mock server written in Go. Register canned responses for any method/path combination, then hit those paths to get the registered responses back.

## Running

```bash
# Default port 8080
docker run -p 8080:8080 kaleman14/mockserver:latest

# Custom port
docker run -e PORT=3000 -p 3000:3000 kaleman14/mockserver:latest
```

The binary also supports a `-healthcheck` flag for container health checks:

```bash
mock-server -healthcheck
```

## Configuration

| Environment Variable | Default | Description |
|---|---|---|
| `PORT` | `8080` | HTTP listen port |
| `FIXTURES_DIR` | `/fixtures` | Directory served under `/__fixtures/` |

## Control API

All control endpoints are under the `/__mock/` prefix.

### `POST /__mock/set`

Register a single mock rule.

**Request body:**

```json
{
  "method": "GET",
  "path": "/api/users",
  "response": {
    "status_code": 200,
    "headers": { "X-Custom": "value" },
    "body": { "users": [] }
  }
}
```

| Field | Type | Default | Description |
|---|---|---|---|
| `method` | string | `POST` | HTTP method to match |
| `path` | string | required | URL path to match |
| `response.status_code` | int | `200` | Response status code |
| `response.headers` | object | none | Response headers |
| `response.body` | any | required | Response body (returned as-is) |
| `response.times` | int | unlimited | Number of times this rule should match before being removed. `0` or omitted means unlimited. |

**Response:**

```json
{ "ok": true, "key": "GET /api/users" }
```

Multiple rules can be registered for the same method+path. They are matched in FIFO order. When a rule with a `times` limit is exhausted, the next rule in the queue is used.

### `POST /__mock/set-many`

Register multiple mock rules in a single request. Takes an array of the same objects as `/__mock/set`.

**Request body:**

```json
[
  { "method": "GET", "path": "/a", "response": { "body": "ok" } },
  { "method": "POST", "path": "/b", "response": { "status_code": 201, "body": { "id": 1 } } }
]
```

**Response:**

```json
{ "ok": true, "count": 2 }
```

### `POST /__mock/set-bulk-decision`

Register a dynamic bulk decision handler. When any request hits a path ending in `/GetDecisionBulk`, this handler generates per-entity decisions instead of using static mock rules.

For each entity in the request, the server extracts the `emailAddress` or `id` field from the entity chain. If the value is in `permit_values`, the decision is `DECISION_PERMIT`; otherwise it uses `default_decision`.

**Request body:**

```json
{
  "permit_values": ["admin@test.com", "user-123"],
  "default_decision": "DECISION_DENY"
}
```

| Field | Type | Default | Description |
|---|---|---|---|
| `permit_values` | string[] | required | Values that receive `DECISION_PERMIT` (matched case-insensitively) |
| `default_decision` | string | `DECISION_DENY` | Decision for non-matching entities |

**Response:**

```json
{ "ok": true }
```

**Example bulk decision request (sent to any path ending in `/GetDecisionBulk`):**

```json
{
  "decisionRequests": [
    {
      "entityIdentifier": {
        "entityChain": {
          "entities": [{ "emailAddress": "admin@test.com" }]
        }
      },
      "resources": [{ "ephemeralId": "res-1" }]
    }
  ]
}
```

**Example bulk decision response:**

```json
{
  "decisionResponses": [
    {
      "resourceDecisions": [
        { "decision": "DECISION_PERMIT", "ephemeralResourceId": "res-1" }
      ]
    }
  ]
}
```

### `DELETE /__mock/reset` or `POST /__mock/reset`

Clear all registered mock rules, the dynamic bulk decision handler, and the request log.

**Response:**

```json
{ "ok": true }
```

### `GET /__mock/log`

Return all recorded requests (both matched and unmatched).

**Response:**

```json
[
  {
    "timestamp": "2025-01-01T00:00:00Z",
    "method": "GET",
    "path": "/api/users",
    "headers": { "Accept": "application/json" },
    "body": null,
    "matched": true
  }
]
```

### `GET /__mock/health`

Health check endpoint.

**Response:**

```json
{ "status": "ok" }
```

## Static Fixtures

`GET /__fixtures/<path>` serves `<FIXTURES_DIR>/<path>` as-is, for responses that cannot be expressed as a JSON mock body (binary files such as app packages). Fixtures are read from disk on each request, so `/__mock/reset` does not affect them. Mount the directory into the container:

```bash
docker run -v "$PWD/fixtures:/fixtures:ro" -p 8080:8080 ghcr.io/rocketchat/mockserver:main
```

`Content-Type` comes from the file extension (`.zip` is `application/zip`), falling back to content sniffing.

## Matching Behavior

Any request that does not hit a `/__mock/*` control endpoint is treated as a mockable request:

1. If the path ends in `/GetDecisionBulk` and a dynamic bulk rule is registered, it is handled dynamically.
2. Otherwise, the server looks up a rule by `METHOD /path`. Rules are matched FIFO.
3. If a matching rule has a `times` limit, it decrements on each use and is removed when exhausted.
4. If no rule matches, the server returns `404` with:

```json
{ "error": "no mock registered", "method": "GET", "path": "/api/users" }
```

All requests (matched or not) are recorded in the request log.

The default `Content-Type` for all responses is `application/json` unless overridden by the mock rule's headers.
