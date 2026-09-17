# Gateway Routing and the MDS Message Standard

- **Status:** Draft
- **Date:** 2026-09-15
- **Scope:** how the Client API Gateway routes a request to Kafka, the message every MDS service exchanges, and the shared Java package that implements both.

---

## 1. At a glance

![One gateway, two modes](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/gwd-01-two-modes.png)

| # | Principle |
|---|---|
| 1 | The mode is set per deployment in the gateway's env: `FORWARD` or `STORAGE` |
| 2 | `FORWARD`: every request goes to `market.forward.request.v1`; the Forward Handler decides everything |
| 3 | `STORAGE`: every request goes to the topic of its business; that business service decides how to answer — its own database, or the trading core |
| 4 | `serviceCode` is the only name for "what to do" — sent by the client in the common API, labelled by the gateway for any other endpoint |
| 5 | One configuration table, one row per `serviceCode` |
| 6 | Every reply returns on one topic, `market.gateway.reply.v1` |
| 7 | One message standard, shipped as a Java package in `libs/common-java` |

---

## 2. From a request to a topic

![From a path to a topic](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/gwd-02-path-to-topic.png)

**Env — per deployment**

| Variable | Example | Meaning |
|---|---|---|
| `GATEWAY_MODE` | `FORWARD` | `FORWARD` or `STORAGE` |
| `GATEWAY_FORWARD_TOPIC` | `market.forward.request.v1` | The one topic of FORWARD |
| `GATEWAY_REPLY_TOPIC` | `market.gateway.reply.v1` | The shared reply topic |
| `GATEWAY_REPLY_PARTITION` | `0` | This instance's partition, from the pod ordinal |
| `GATEWAY_CONFIG_REFRESH` | `30s` | How often the table is re-read |
| `GATEWAY_MAX_BATCH_ITEMS` | `100` | Largest common-API batch |
| `GATEWAY_MAX_IN_FLIGHT` | `5000` | Messages awaiting a reply; above it the gateway answers `503` |

**When a step fails**

| Step | Result |
|---|---|
| No row for the path | `404`, nothing published |
| Malformed batch | `400`, nothing published |
| Item `serviceCode` not a row of this path | Item error `REQUEST_SERVICE_NOT_ALLOWED` |
| Row not `ACTIVE` · invalid `args` · missing credential | `REQUEST_SERVICE_SUSPENDED` · `REQUEST_INVALID_ARGS` · `401 AUTH_TOKEN_MISSING` |
| `STORAGE` and the row has no topic | `REQUEST_SERVICE_UNAVAILABLE` |
| Kafka unavailable | `503 UPSTREAM_KAFKA_UNAVAILABLE` |

---

## 3. The route table

![One table, one row per serviceCode](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/gwd-03-route-table.png)

```sql
CREATE TABLE mds.gateway_route (
    service_code     varchar(80)  PRIMARY KEY,   -- chart · auth.login · package.subscribe
    http_method      varchar(10)  NOT NULL,      -- POST
    path             varchar(200) NOT NULL,      -- /api/v1/common/market · /api/v1/auth/login
    dispatch         varchar(10)  NOT NULL,      -- BATCH: serviceCode from the body · SINGLE: this row is the endpoint
    storage_topic    varchar(120),               -- STORAGE only; NULL = not served in STORAGE
    key_strategy     varchar(30)  NOT NULL,      -- CORRELATION_ITEM | REQUEST_ID | ARG | SUBJECT
    key_arg          varchar(60),                -- the args field for ARG, e.g. user
    credential_rule  varchar(30)  NOT NULL,      -- NONE | MDS_AUTH_REQUIRED
    args_schema_ref  varchar(120) NOT NULL,      -- JSON Schema of args
    timeout_ms       integer      NOT NULL DEFAULT 5000,
    status           varchar(20)  NOT NULL DEFAULT 'DRAFT',   -- DRAFT | ACTIVE | SUSPENDED | RETIRED
    description      varchar(200),
    version          bigint       NOT NULL DEFAULT 1,
    updated_at       timestamptz  NOT NULL DEFAULT now(),
    updated_by       varchar(60),
    CHECK (dispatch IN ('BATCH', 'SINGLE')),
    CHECK (key_strategy <> 'ARG' OR key_arg IS NOT NULL)
);
CREATE INDEX gateway_route_path ON mds.gateway_route (http_method, path);
```

**Rules checked when the table is loaded** — a row breaking one is rejected

| # | Rule | Why |
|---|---|---|
| 1 | All rows of one path share the same `dispatch` | A path is either a batch or a single call |
| 2 | A `SINGLE` path has exactly one `ACTIVE` row | The gateway must know which `serviceCode` to label |
| 3 | A `serviceCode` is reachable only through its own path | `auth.login` cannot be sent inside the common market batch |
| 4 | `storage_topic` is named `market.<domain>.<name>.request.v<major>` | One naming convention |

**Seed — HK / US FORWARD deployment** (`storage_topic` is filled so the same rows serve a STORAGE deployment)

```sql
INSERT INTO mds.gateway_route
  (service_code, http_method, path, dispatch, storage_topic, key_strategy, key_arg,
   credential_rule, args_schema_ref, timeout_ms, status) VALUES
  ('symbolatest', 'POST', '/api/v1/common/market', 'BATCH', 'market.query.snapshot.request.v1',
   'CORRELATION_ITEM', NULL, 'MDS_AUTH_REQUIRED', 'SymbolAtestArgs', 5000, 'ACTIVE'),
  ('chart',       'POST', '/api/v1/common/market', 'BATCH', 'market.query.timeseries.request.v1',
   'CORRELATION_ITEM', NULL, 'MDS_AUTH_REQUIRED', 'ChartArgs', 5000, 'ACTIVE'),
  ('company',     'POST', '/api/v1/common/market', 'BATCH', 'market.query.reference.request.v1',
   'CORRELATION_ITEM', NULL, 'MDS_AUTH_REQUIRED', 'CompanyArgs', 5000, 'ACTIVE'),
  -- one row per market data service
  ('auth.login',  'POST', '/api/v1/auth/login',    'SINGLE', 'market.auth.request.v1',
   'ARG', 'user', 'NONE', 'LoginArgs', 10000, 'ACTIVE');
```

**Adding the package APIs** — three rows; no code, no new topic

```sql
INSERT INTO mds.gateway_route
  (service_code, http_method, path, dispatch, storage_topic, key_strategy,
   credential_rule, args_schema_ref, timeout_ms, status) VALUES
  ('package.list',      'GET',  '/api/v1/packages',           'SINGLE', 'market.package.request.v1', 'REQUEST_ID', 'MDS_AUTH_REQUIRED', 'PackageListArgs',      5000,  'ACTIVE'),
  ('package.subscribe', 'POST', '/api/v1/packages/subscribe', 'SINGLE', 'market.package.request.v1', 'SUBJECT',    'MDS_AUTH_REQUIRED', 'PackageSubscribeArgs', 10000, 'ACTIVE'),
  ('package.cancel',    'POST', '/api/v1/packages/cancel',    'SINGLE', 'market.package.request.v1', 'SUBJECT',    'MDS_AUTH_REQUIRED', 'PackageCancelArgs',    10000, 'ACTIVE');
```

For a `GET`, `args` come from the query string. `REQUEST_ID` uses the client's `Idempotency-Key` header when one is sent.

---

## 4. Where the table comes from

![Where the table comes from](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/gwd-04-config-source.png)

| Phase | Source |
|---|---|
| Interim | The gateway reads `mds.gateway_route` at startup and every `GATEWAY_CONFIG_REFRESH`. A refresh that breaks a rule keeps the previous snapshot and raises an alert |
| Target | admin-service publishes each row on the compacted topic `market.config.gateway-route.v1`, key `serviceCode` |

---

## 5. The message

![One envelope for every service](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/gwd-05-envelope.png)

**Request** — a login, and one common-API item

```json
{ "header": {
    "messageId": "9f2c…", "messageType": "ServiceRequest", "schemaVersion": 1,
    "correlationId": "5b1e…", "causationId": "5b1e…",
    "occurredAt": "2026-09-15T03:10:00Z", "deadlineAt": "2026-09-15T03:10:10Z",
    "replyTo": { "topic": "market.gateway.reply.v1", "partition": 3 },
    "source": "client-api-gateway/3", "traceId": "…" },
  "payload": { "serviceCode": "auth.login", "requestId": "a71e…", "args": { "user": "uat_qw02" } } }
```

```json
{ "header":   { "messageType": "ServiceRequest", "correlationId": "77d0…", "…": "…" },
  "security": { "providerToken": "<MDS-Authentication>" },
  "payload":  { "serviceCode": "chart", "itemIndex": 1,
                "args": { "exchange": "XHKG", "symbol": "00700", "period": "1D" } } }
```

**Reply**

```json
{ "header":  { "messageId": "c41a…", "messageType": "ServiceReply", "schemaVersion": 1,
               "correlationId": "77d0…", "causationId": "<request messageId>",
               "occurredAt": "…", "source": "forward-handler/1" },
  "payload": { "serviceCode": "chart", "itemIndex": 1, "isSuccess": false,
               "error": { "code": "UPSTREAM_TIMEOUT", "message": "Provider did not answer",
                          "retryable": true } } }
```

Error codes and their retryability follow the platform catalogue: `REQUEST_*`, `AUTH_*`, `ENTITLEMENT_*`, `UPSTREAM_*`, `STORAGE_*`, `TIMEOUT_*`, `INTERNAL_*`.

---

## 6. Replies

![A batch, answered out of order](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/gwd-06-batch-assembly.png)

![One reply topic, one partition per gateway](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/gwd-07-reply-partitions.png)

| Event | Handling |
|---|---|
| Reply arrives | Its `causationId` is in `waiting` → it fills `slots[itemIndex]` |
| `waiting` empty | The gateway answers at once |
| `deadlineAt` reached | Unanswered slots get `TIMEOUT_DOWNSTREAM` |
| Late, duplicate or unknown reply | Discarded and counted |

---

## 7. The shared package — `com.grooo.common.messaging`

![What the shared package does](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/gwd-08-shared-package.png)

| Component | Role |
|---|---|
| `Envelope`, `ServiceRequest`, `ServiceReply`, `ServiceError` | The message of §5 |
| `EnvelopeCodec` | JSON, unknown fields ignored, header validated on read |
| `Keys` | The key strategies of §2 — a producer cannot invent one |
| `RequestClient` · `BatchCall` | Caller side: publish, wait on the own partition, match by `causationId`, enforce the deadline, bound in-flight calls, assemble a batch |
| `ServiceRouter` · `@ServiceHandler` | Receiver side: consume, drop expired requests, dispatch by `serviceCode`, reply exactly once, map exceptions to errors |
| `ErrorCodes` | The error catalogue with retryability |
| `Redactor` · `DeadLetterPublisher` | Mask `security` in logs; strip it from dead letters |
| Core bridge | Converts to and from the trading core's message format |
| Spring Boot auto-configuration | `grooo.messaging.*` properties → a ready `RequestClient` or `ServiceRouter` |

**What a developer writes — a business service**

```java
@ServiceHandler("package.subscribe")
public SubscriptionResult subscribe(PackageSubscribeArgs args, RequestContext ctx) {
    // the service decides: its own database, or the trading core through the core bridge
    return packageService.subscribe(ctx.subjectRef(), args.packageCode(), ctx.requestId());
}
```

```yaml
grooo.messaging:
  consume:
    topic: market.package.request.v1
    group: mds.package
```

**What a developer writes — the caller**

```java
CompletableFuture<ServiceReply> reply = requestClient.send(ServiceCall.builder()
    .topic(topic).key(key)
    .serviceCode("chart").itemIndex(1).args(args)
    .deadline(deadlineAt).security(security)
    .build());
```

---

## 8. Open decisions

| # | Question | Proposal |
|---|---|---|
| 1 | Naming of business `serviceCode`s | `<domain>.<action>` — `auth.login`, `package.subscribe`. Market data codes keep their published names |
| 2 | STORAGE topic names | Market data: `market.query.{snapshot,timeseries,reference}.request.v1`. Businesses: `market.auth.request.v1`, `market.package.request.v1` |
| 3 | Route per exchange | Not needed — one mode per deployment |
| 4 | `SUBJECT` key in FORWARD, where the gateway reads no token | Falls back to `REQUEST_ID` |
