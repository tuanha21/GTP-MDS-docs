# Market Data Server — System Specification

- **Status:** Draft, under construction
- **Date:** 2026-09-11
- **Related:** `market-data-schema-design.md`, `market-data-flow-design.md`, `market-data-server-system-specification.archived-2026-09-11.md` (previous version)

---

## How to read this document

| Part | Content | Written for |
|---|---|---|
| **A — Overview** | What the system is, its boundaries, the shared models. Diagrams **by layer** | Everyone |
| **B — Functional design** | Each capability end-to-end across services: entry point, request split, envelopes, topics, result assembly. Diagrams **by flow** | Integrators, BA, QA |
| **C — Service design** | Each service internally. Diagrams **by service** | Engineers |
| **D — Cross-cutting** | Data model, topics, security, reliability, open decisions | Reference |

A capability crosses several services, so Part B carries the whole picture that Part C cannot.

Part B section shape: *purpose → actors and entry point → flow diagram → steps with envelopes and topics → error cases → participating services*.

| Marker | Meaning |
|---|---|
| **Settled** | Decided. Build against this |
| **Drafted** | Written, not yet confirmed |
| **Open** | Decision required, with the owner named |

Design detail is specified **structurally**: for data, which attributes are essential and which are left to implementation; for code, interfaces and their function, not classes.

---

## Document status

Progress at a glance. **Settled** sections are safe to build against; **Open** sections have not been designed yet.

**Current priority: the FORWARD deployment.** Content that applies only to STORAGE — entitlement, subscription, licence reporting — is kept and marked, but deferred.

| § | Section | State |
|---|---|---|
| A1 | What MDS is | Settled |
| A2 | Responsibility boundaries | Settled |
| A3 | System architecture | Drafted |
| A4 | Service map | Drafted |
| A5 | Identity and token model | Drafted |
| A6 | Commercial model | Drafted · A6.4 open |
| A7 | Offering and entitlement model | Settled |
| A8 | Connection mode — one per deployment | Settled |
| A9 | Realtime and delayed delivery | Settled |
| A10 | Key constraints | Drafted |
| B1 | Authentication | Drafted |
| **B2** | **Package, subscription and entitlement** | **Open** |
| B3 | Common Query API | Drafted |
| B4 | Streaming subscription | Drafted |
| **B5** | **Delayed delivery** | **Open** |
| **B6** | **Market data ingestion (STORAGE)** | **Open** |
| B7 | Forward request (FORWARD) | Drafted |
| **B8** | **Offering and package administration** | **Open** |
| B9 | Licence reporting | Drafted |
| B10 | Provider integration and the adapter layer | Drafted |
| B11 | Service communication and topics | Drafted |
| C1 | Login relay — a Forward Handler service code | Drafted |
| C2 | Client API Gateway | Drafted |
| C3 | Token reading | Deferred |
| C4 | Subscription & Entitlement Service | Drafted · outline only |
| C5 | Route & Offering Control | Drafted |
| **C6** | **Query Services** | **Open** |
| C7 | Forward Handler and Provider Adapter | Drafted |
| C8 | Ingestion and the Provider SPI | Drafted · 12 subsections |
| **C9** | **Processing and Delayed Delivery** | **Open** |
| C10 | Streaming Distribution | Drafted |
| **C11** | **Admin Services** | **Open** |
| D1 | Data Model | Drafted · 14 clusters · D1.18 open |
| D2 | Kafka Topics and Envelopes | Drafted |
| **D3** | **Security** | **Open** |
| D4 | Reliability and physical design | Drafted · D4.2 open |
| D5 | Licence Reporting | Drafted |
| D6 | Technology Baseline | Settled |
| D7 | Open Decisions | 15 outstanding |

Not yet started: **8 of 39 sections** — B2, B5, B6, B8, C6, C9, C11, D3.

---

# Part A — System Overview

## A1. What MDS is — **Settled**

MDS is a **self-contained market-data product**. It integrates external providers, normalizes their payloads into one canonical model, stores what the licences permit, defines and sells its own packages, and serves snapshot, historical and streaming data through one client contract that does not change with the provider behind it.

| Property | Meaning |
|---|---|
| Provider-agnostic core | Provider codes, symbols and wire protocols exist only in the integration layer and mapping tables |
| Core-agnostic commerce | MDS owns its package catalogue and subscription state; a trading core integrates with MDS |
| Login Server as the entrance | The client logs in at the Login Server and presents its token to MDS. MDS issues no token and stores none |
| One internal transport | Services communicate only through Kafka |
| One mode per deployment | A deployment runs FORWARD or STORAGE, never both (§A8) |
| FORWARD is pass-through | In FORWARD the gateway validates no token; it routes, and the handler maps the provider's answer to the platform API |

> **Design intent.** MDS must be deployable against a different trading core, or none at all, without redesign.

## A2. Responsibility boundaries — **Settled**

| MDS owns | MDS does not own |
|---|---|
| Provider integration, normalization, canonical model | Login and token issuance — the Login Server |
| STORAGE / FORWARD route selection per market | Pricing, tax, invoicing, payment collection |
| Ingestion, processing, storage, distribution | Orders, executions, balances, positions |
| Technical offering definitions | The customer's commercial relationship and support |
| Commercial package definitions | |
| Customer subscription state | |
| Routing and response mapping of forwarded requests | |
| Entitlement resolution and enforcement — STORAGE deployments | |
| Per-customer access records for licence reporting — STORAGE deployments | |

### A2.1 Source of trust

| Information | Source | Note |
|---|---|---|
| Technical offering | **MDS** | Define, version, enforce |
| Package definition | **MDS** | Composition of offerings; no price |
| Subscription state | **MDS** | MDS *is* the subscription master |
| Effective entitlement | **MDS** in STORAGE · **the provider** in FORWARD | FORWARD relies on the provider checking the customer's own token |
| User identity | **Login Server** — interim: TTL core trading login | MDS issues no identity |
| Price, tax, payment outcome | Trading core | MDS requests a charge; the core collects |
| Provider market data | Provider | Normalize and retain where licensed |
| Realtime access records | **MDS** in STORAGE · the provider in FORWARD | FORWARD calls arrive under the customer's own identity |

### A2.2 Trade-off of this boundary

| Gain | Cost |
|---|---|
| Entitlement is local — no cross-system synchronization, no propagation lag | MDS carries commercial surface: subscription lifecycle, immutable audit, reconciliation with the core's payment records |
| Revocation is a local write, effective immediately | MDS must expose subscription APIs and a charge contract that any core can implement |
| MDS is deployable against any core, or standalone | More to build than a pure data plane |
| Licence declarations derive from subscription state, not inferred from traffic | |

## A3. System architecture — **Drafted**

![MDS platform architecture from client to provider](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-platform-architecture.png)

**Text alternative:** A customer application (MTS, WTS) first logs in at the Login Server, outside MDS, then reaches shared NGINX with the token it received; an Admin Web application also reaches NGINX. NGINX routes to the Client API Gateway or the Admin API Gateway. From the gateway all work crosses the single MDS Kafka backbone. Platform services attach to that backbone: the interim login relay, commerce and entitlement, route and offering control, processing and distribution, storage, and the provider integration layer — the only vendor-aware code — which reaches external providers such as TTL (including the TTL MDS login in the interim), ICE and the trading core.

| # | What the diagram asserts |
|---|---|
| 1 | Customer and admin are separate paths, separate gateways, separate contracts |
| 2 | NGINX is shared and sits outside the MDS boundary |
| 3 | The gateway is the entry boundary — it routes every request; in STORAGE it decides access, in FORWARD the provider does |
| 4 | Kafka is the only internal transport, so any service scales, is replaced or replays independently |
| 5 | The provider integration layer is the only vendor-aware code — the extension point per provider |
| 6 | The Login Server is the one external system the client reaches — to log in. Providers stay outside the boundary; the client never calls them directly |
| 7 | Each deployment runs one connection mode — FORWARD or STORAGE (§A8) |

## A4. Service map — **Drafted**

| Service | Owns | Does not own |
|---|---|---|
| **Login relay** — `serviceCode auth.login` of the Forward Handler (§C1) | The customer's TTL MDS login: encrypts the mapped user id and a timestamp, returns TTL's session unchanged. Interim, until the Login Server is live | Token issuance, signing, storage, entitlement |
| **Client API Gateway** | Contract validation, routing, token pass-through in FORWARD, correlation, deadlines, field projection | Token issuance or validation in FORWARD, provider protocol, storage |
| **Admin API Gateway** | Admin contract, permission claims, maker-checker entry | Business mutation logic |
| **Subscription & Entitlement** | Package catalogue, subscription state, entitlement resolution, charge requests — STORAGE deployments | Price, tax, payment outcome |
| **Route & Offering Control** | Offering catalogue; licence state; which services a deployment may serve — **control plane, not on the request path** | Per-request routing, provider execution |
| **Query · Snapshot** | Latest quote and order book | History, reference |
| **Query · Time-series** | Bars and history across the tier chain | Snapshot semantics |
| **Query · Reference** | Instrument, company, fundamentals, corporate actions | Price data |
| **Forward Handler** — `forward-service` | Provider-routed requests with the customer's own provider token, mapping to the platform API, multiplexing | Long-term retention, entitlement |
| **Ingestion** | Provider feeds via the adapter SPI, sequence and freshness validation | Client authorization |
| **Processing** | Normalization, aggregation, derived calculation | Entitlement |
| **Delayed Delivery** | Holding and releasing events no earlier than the configured delay | Claiming delay without enforcing it |
| **Streaming Distribution** | Authorized subscription fan-out | Login or subscription ownership |
| **Storage Writers** | Persistence per retention policy | Unbounded retention |
| **Admin Services** | Catalogue, package, provider, route, mapping, operations | Direct calls from Admin Web |

### A4.1 Why the query path is three services — **Settled**

Split by **SLA**, not by storage technology.

| Service | Typical QPS | Target p99 | Stores |
|---|---|---|---|
| Snapshot | Very high | < 20 ms | Redis |
| Time-series | Medium | < 200 ms | Redis → PostgreSQL → ClickHouse |
| Reference | Low | < 500 ms | PostgreSQL |

| Gain | Cost |
|---|---|
| A five-year cold query cannot stall the live price board — separate thread pools and scaling | Three deployables instead of one |
| Each service scales on its own load curve | A client batch touching all three fans out wider |

The tier chain lives **inside** the time-series service, driven by retention configuration, so tier boundaries move as configuration.

## A5. Identity and token model — **Drafted**

### A5.1 The Login Server is the entrance

![The Login Server is the front door](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-a5-login-entrance.png)

**Text alternative:** The trading app logs in at the Login Server, which is outside MDS and owned by HK, and receives a token. Every call the app makes to MDS carries that token in its header. The MDS API Gateway reads the header, validates nothing, and routes the request with the token to the Forward Handler, which calls the provider. The provider checks the token and the customer's rights, and the answer, mapped to the platform API, returns to the app. MDS issues no token and stores none.

The client logs in at the Login Server — not at MDS — and attaches the token to every call. MDS issues no token and keeps none; in a FORWARD deployment it validates none either.

| Who | Does | Does not |
|---|---|---|
| **Login Server** — HK, outside MDS | Authenticates the user; issues the token (JWE planned) | Take part in the request path |
| **Client** | Logs in; attaches the token to every call; renews it with the issuer | Call a provider directly |
| **MDS gateway** | Routes the request; in FORWARD passes the token on unread | Issue, sign, validate or store a token |
| **Provider** — FORWARD | Checks the token and the customer's rights | — |

| Gain | Cost |
|---|---|
| MDS runs no customer login, no signing key and no token check | MDS follows the Login Server's token contract, not yet published (D18) |
| The request path never calls another system and never decodes a token | A bad token is found by the provider, one hop later |

### A5.2 Two phases

| Phase | What the client sends on every MDS call | Where it comes from |
|---|---|---|
| **Interim** — now, until the Login Server reaches UAT | `Authorization` — TTL core trading token · `MDS-Authentication` — TTL MDS session id (`sessID`). In the body when a header cannot be set (§B7.3) | TTL core trading login; TTL MDS login, performed by the MDS login relay (§C1) |
| **Target** — Login Server live | Per the Login Server (D18) | The Login Server |

Switching is a change to which value the gateway forwards. Nothing past the gateway reads a token.

### A5.3 MDS never stores credentials or tokens — **Settled**

MDS stores no password and no token in a database, a cache, a log, a trace or a dead-letter record. The one place a token travels inside MDS is the forward request topic, which carries the provider token to the Forward Handler (§B7.5).

<mark>Forwarding a message through Kafka is itself a form of storage.</mark> The forward topic keeps a message for one minute.

### A5.4 Token parameters — **Drafted**

| Parameter | Value |
|---|---|
| Token lifetime | Set by the issuer — TTL now, the Login Server later |
| Refresh | Between the client and the issuer. MDS has no refresh endpoint; renewing the TTL MDS session is a new login through the relay |
| Reading the token | Not in FORWARD. Later, if ever needed: decrypt the Login Server's JWE with its key (§C3) |

## A6. Commercial model — **Drafted**

### A6.1 Three levels

```
Technical offering    An atomic right.   MDS:HK:QUOTE:L1:STREAM:REALTIME
        ↓ composed into
Package               What is sold.      "HK Professional"
        ↓ held by
Subscription          Who holds it, from when, in what state
```

MDS owns all three. The trading core owns **price, tax and collection**.

### A6.2 MDS holds no price — **Settled**

| Gain | Cost |
|---|---|
| MDS stays out of jurisdiction-specific pricing and tax law, and out of financial audit scope | A charge request must name the package and period; the core resolves the amount |
| Package definition changes independently of commercial pricing | Reconciliation between MDS subscription state and core payment records is required |

### A6.3 Subscription lifecycle — **Settled** for phase one

```
PENDING → ACTIVE → CANCELLED
                 ↘ EXPIRED
                 ↘ (renew) → ACTIVE
```

| Gain | Cost |
|---|---|
| Small enough to build and test within the phase-one window | Trials, mid-cycle upgrade and downgrade are not available at launch |
| Later states extend the model without breaking it | Proration stays with the core, so mid-cycle changes need a core-side answer |

### A6.4 Payment failure — **Open (D8), business decision**

When the core reports a failed payment, what happens to access is commercial policy.

| Option | Behaviour | Consequence |
|---|---|---|
| **Grace period, then cut** | `PAST_DUE` retains access for N days, then suspended | Best experience. **A customer in `PAST_DUE` still counts toward the exchange declaration — licence fees continue** |
| **Cut immediately** | Access ends on reported failure | Lowest licence cost and risk. A transient failure such as an expired card cuts a paying customer |
| **Fall back to a free tier** | Move to a basic or delayed package | Retains the customer. Must be an **explicit package change**, never a silent downgrade (§A7) |

**Recommendation:** model the grace period as a per-package parameter so policy is configuration; the customer sets the default. The design supports all three.

### A6.5 Charge contract — **Drafted**

| Direction | Content |
|---|---|
| MDS → core | customer, package code, billing period, subscription reference |
| Core → MDS | payment outcome: succeeded, failed, reason |

MDS never calls the core synchronously and never learns the amount. Any core satisfying this two-message contract can be integrated.

## A7. Offering and entitlement model — **Settled**

> Applies to STORAGE deployments. In a FORWARD deployment the provider enforces entitlement (§A8).

![How an atomic MDS offering is constructed](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-access-modes.png)

| Layer | Dimension | Example |
|---|---|---|
| 1 — Data scope | Market | `HK`, `KR`, `US`, `CN` |
| 1 — Data scope | Dataset | quote, order book, trade, chart |
| 1 — Data scope | Market depth | `L1`, `L2`, full book |
| 2 — Access method | Snapshot or stream | `SNAPSHOT`, `STREAM` |
| 3 — Delivery timing | Realtime or delayed | `REALTIME`, `DELAYED:900` |
| Cross-cutting | Route, licence, lifecycle | Provider route, effective dates, status |

Non-inheritance rules, all enforced:

- A snapshot right never grants streaming
- A delayed-stream right never grants realtime
- An entitlement for one exchange never grants another, even for the same symbol
- MDS never silently downgrades realtime to delayed, or stream to snapshot

### A7.1 Entitlement is per customer — **Settled**

| Gain | Cost |
|---|---|
| Matches how exchanges count licences — one natural person, not one per account | Request context must carry the account separately for audit |
| A customer with three accounts is one licensed user, so no over-reporting or over-paying | Account-scoped offerings, if ever needed, require an explicit extension |

## A8. Connection mode — one per deployment — **Settled**

![Per-market storage and forward connection modes](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-market-connection-modes.png)

Each deployment runs **exactly one** mode. No deployment serves the same request both ways.

| Mode | Behaviour | Status |
|---|---|---|
| **FORWARD** | Passes the client's request and token to the provider and maps the response to the platform API — fields present are mapped, fields absent are left empty. Validates no token, checks no entitlement, stores nothing, keeps no access record. Hong Kong and US, via TTL | **Current priority** |
| **STORAGE** | Ingests, normalizes, stores, processes, distributes. MDS enforces entitlement from its own subscription state. Requires storage and redistribution rights. Korea, via ICE | Deferred |
| **DISABLED** | Rejects an intentionally unavailable operation | — |

**Connection mode decides who enforces entitlement.** In STORAGE, MDS does (§A7). In FORWARD, the provider does, against the customer's own token (§C7.1).

## A9. Realtime and delayed delivery — **Settled**

Only the **latest snapshot** and **live push** need a separate delayed path. Completed historical bars are identical for both audiences; the difference is only how recent a bar may be seen.

| Data | Delayed treatment |
|---|---|
| Historical bars | Single store; filter `bar_time <= now() - delay` at query time |
| Latest snapshot | Separate Redis namespace `md:dl:*` |
| Live push | Separate subscriber set fed from the delayed namespace |

The delayed path is a **second consumer group on the same Kafka topics**, pausing its partitions while the head record is younger than the configured delay.

| Gain | Cost |
|---|---|
| Kafka is the buffer — no republished topic, no in-memory queue, no duplicated history | A second consumer group to operate and monitor for lag |
| Realtime and delayed live in **structurally separate stores**, so separation is enforced by infrastructure, not by an application condition | Snapshot state is duplicated (~31 MB at ~31,000 instruments) |

## A10. Key constraints — **Drafted**

| Constraint | Why it matters |
|---|---|
| Provider capability and licence confirmed per market | MDS must not advertise an unsupported or unlicensed combination |
| Delayed release technically enforced | Early release is a licensing incident, not a latency defect |
| Realtime access recorded per customer — STORAGE | In FORWARD the provider holds the record, since each call carries the customer's own token |
| Customer classification available — STORAGE | Exchanges price and count retail and professional separately |
| Route fallback preserves semantics | Realtime must never be silently downgraded |

---

# Part B — Functional Design

> Each section describes one capability end-to-end across services.

## B1. Authentication — **Drafted**

### B1.1 Purpose

**The Login Server is the entrance to the whole system.** The client logs in there and attaches what it receives to every call. MDS issues no token and keeps none. In a FORWARD deployment it **validates none either**: FORWARD only routes the request and maps the provider's answer to the platform API, and the provider checks the token. Overview in §A5; the forward path in §B7.

Until the Login Server reaches UAT, the client holds two TTL tokens (§B1.3).

### B1.2 Actors, entry points and credentials

| | Interim — now | Target |
|---|---|---|
| Login | 1 · TTL core trading login — password, 2FA, captcha — returns token 1 and the TTL MDS user mapped to the account. 2 · `POST /api/v1/auth/login` on the MDS login relay with `{ "user": … }` → `mdsToken`, the TTL MDS session id (§B7.2) | The Login Server's own login |
| Renewal | Repeat the login that issued the expiring token | Per the Login Server |
| On every MDS call | `POST /api/v1/common/market` with `MDS-Authentication: <mdsToken>`; `Authorization: Bearer <token 1>` passed as sent. When a header cannot be set, the same values go in the body (§B7.3) | Per the Login Server (D18) |

### B1.3 Flow — interim

![Interim login — two TTL tokens](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-b1-interim-login.png)

**Text alternative:** The client logs in to TTL core trading — with password, 2FA and captcha where TTL requires them — and receives token 1 together with the TTL MDS user mapped to that account. It sends that user to MDS, whose login relay — an operation of the Forward Handler — encrypts the user id and a timestamp, logs in to TTL MDS, and returns TTL's session id unchanged as token 2. Every call to MDS carries both. The gateway validates neither; it routes the request, and token 2 travels on the forward topic to the Forward Handler, which calls TTL MDS with it and maps the answer to the platform API.

| # | From → to | What the destination is responsible for |
|---|---|---|
| **1** | Client → TTL core trading login | The normal client login, with 2FA and captcha where TTL requires them. Returns token 1 and the TTL MDS user mapped to the account |
| **2** | Client → MDS login relay | Through the gateway, as `serviceCode auth.login` of the Forward Handler (§B11.6); takes the mapped user as sent (§C1.2) |
| **3** | Login relay → TTL MDS login | Sends `LOGIN` with `hex(AES(entity:UTC timestamp:user, MDS key))` — no password, no second factor — and returns TTL's `sessID` to the client **unchanged** as token 2. TTL keeps about one session per user: a new login ends the previous one |
| **4** | Client → Gateway | Every call carries both tokens |
| **5** | Gateway → Forward Handler | No token is validated. The request is routed, and token 2 rides the forward topic as `providerToken` (§B7.5) |
| **6** | Forward Handler → TTL MDS | Sends TTL's message with `sessID` = token 2; TTL checks it. The handler maps the answer to the platform API and drops the token |

### B1.4 FORWARD passes the token through — **Settled**

| The gateway | |
|---|---|
| Does | Check `MDS-Authentication` is present · route the request · put it on the forward request as `providerToken` |
| Does not | Decode, verify, decrypt or cache a token · check entitlement · read `Authorization` |
| Why | FORWARD only stands between the client and the provider: it routes, and maps the provider's API to the platform's. The provider already checks the token; checking it again adds cost and no safety |

### B1.5 The provider token on the forward topic

The provider token is the one exception to the rule that no token enters a Kafka message.

| Rule | |
|---|---|
| Topic | Only `market.forward.request.v1`, as `security.providerToken` (§B7.5) |
| Never in | a log, a trace, a dead-letter record, a retry diagnostic, an error message, a reply |
| Forward Handler | Uses it for the TTL call, then drops it |
| Other handlers | Never receive it — they do not consume the forward topic (§B3.6) |
| Retention | One minute |

<mark>Forwarding a message through Kafka is itself a form of storage: the token sits on the broker's disk until retention removes it.</mark>

### B1.6 Renewal

The client renews with whoever issued the expiring token — TTL core trading, the MDS login relay, or later the Login Server — and sends the new token on its next call. On an open stream the new token is sent in-band on the same connection (§C10.2).

### B1.7 Error cases

| Case | Result |
|---|---|
| `MDS-Authentication` missing | `AUTH_TOKEN_MISSING`; nothing dispatched |
| TTL MDS rejects the session — expired or invalid | `AUTH_TOKEN_EXPIRED`, mapped from TTL's `rtnCode 401`; the client logs in again |
| TTL MDS rejects the relayed login | Login refused; reported without TTL's own error text |
| TTL MDS login unreachable | Login refused. Sessions already issued keep working until they expire |
| Other forward errors | §B7.7 |

### B1.8 Participating services

| Service | Role | Detail |
|---|---|---|
| Forward Handler — `auth.login` | Steps 2–3 | §C1 |
| Client API Gateway | Steps 4–5 | §C2, §B7 |
| Forward Handler | Step 6 | §C7 |

## B2. Package, subscription and entitlement — **Open**

Will cover: package composition from technical offerings, subscription grant and renewal, the charge exchange with the trading core, and entitlement resolution at request time. Applies to STORAGE deployments.

Design decisions already settled in §A6; data model in §D1.3. Blocked on **D8** (payment-failure access policy).


## B3. Common Query API — **Drafted**

### B3.1 Purpose

![One call, many answers](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-b3-query-overview.png)

**Text alternative:** A trading app whose screens each need different numbers sends one request listing what it wants. The API Gateway checks who is asking and what they are allowed to see, splits the request, and hands each part to the service that owns that kind of data — prices from the cache, charts from the time-series store, financials from the database. The answers come back as one response in the order asked.

A trading screen is several different things at once: a price board, the detail of one stock, a chart, a company's financials. Each is held in a different place and fetched a different way. The app does not make one call per piece — it makes **one call that lists what it wants**, and gets one reply back in the same order.

| | One endpoint per kind of data | One common endpoint |
|---|---|---|
| Calls needed to fill a screen | one per kind of data | one |
| Permission checks | one per call | one, for the whole batch |
| Time to fill the screen | the client waits on each call it makes | the pieces are fetched at the same time |
| One piece failing | the client handles each failure separately | that slot carries an error; the rest still arrive |
| Adding a new kind of data | a new endpoint and a client release | a new catalogue entry, no client release (§B3.8) |

Stated precisely: one endpoint serves every read. A client sends a batch of named service calls and states which fields it wants back. The batch is **N independent requests sharing one HTTP round trip** — not one combined permission, and not a transaction.

### B3.2 Actors and entry point

| | |
|---|---|
| Actor | Customer application (MTS, WTS) carrying the headers in §B1.2 |
| Entry point | `POST /api/v1/common/market` |
| Body | Array of 1..N service calls — or `{ "auth": {…}, "items": [ … ] }` when credentials cannot go in headers (§B7.3) |
| Response | Array of the same length and order — correlation is **by position** |

| Item field | | Meaning |
|---|---|---|
| `serviceCode` | Essential | Must exist and be ACTIVE in the service catalogue |
| `args` | Essential | Validated against that service's argument schema |
| `projection` | Optional | Top-level field names; empty or absent returns all |

**Two identifiers carry the batch through the system.** Neither is supplied by the client.

| Identifier | Where it comes from | Purpose |
|---|---|---|
| `correlationId` | Generated by the gateway, one per HTTP request | Ties every message of one batch together, across every hop and every log line |
| `itemIndex` | **Not generated — it is the item's zero-based position in the request array** | Identifies which item a reply answers, and which slot of the response array it fills |

`itemIndex` is already the response correlation key, because the response array has the same length and order as the request. Using it in the partition key reuses an identifier that has to exist anyway rather than inventing a second one.

**Why the key is the pair, not `correlationId` alone.** Keying on the batch would send every item of a batch to the same partition, so one consumer would work through them in series — defeating the point of independent items. Adding `itemIndex` gives each item its own key, so a batch fans out across partitions and its items run in parallel. Since `correlationId` is a fresh UUID per batch, the pair distributes evenly with no further tuning.

### B3.3 Flow

![Common Query API request and reply flow](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-common-query-flow.png)

**Text alternative:** A client posts a batch to NGINX, which routes it to an API Gateway instance. The gateway reads the headers and, in FORWARD, passes the provider token through without validating it; it resolves each item's serviceCode against its cached gateway_route snapshot and publishes one envelope per allowed item straight to the topic that row names. Each handler consumes its own topic, reads its own store or calls a provider, and publishes a reply keyed by the waiting gateway instance. A query costs two Kafka messages per item and no token check.

| # | From → to | Transport | What the destination is responsible for |
|---|---|---|---|
| **1** | Client → NGINX | HTTPS | Terminate TLS · route by path to the MDS or TS gateway · load balance across gateway instances · coarse per-IP and body-size limits from static config. **No awareness of tokens, customers or entitlement** |
| **2** | NGINX → Client API Gateway | HTTP | Everything in §B3.4 |
| **3** | Gateway → Kafka | destination topic chosen in step 2 | **One envelope per item**, whatever the item's symbol count. Key `correlationId:itemIndex` |
| **4** | Kafka → Handler | that handler's own consumer group | Deserialize payload by `messageType` · **never re-check entitlement** — the authorized scope is in the envelope · check `deadlineAt` and abandon rather than execute if past |
| **5a** | Query handler ↔ store | — | Snapshot reads Redis; time-series walks hot → warm → cold; reference reads PostgreSQL |
| **5b** | Forward handler ↔ provider | vendor protocol | Resolve the upstream session, call the provider, normalize the response |
| **6** | Handler → Kafka | `market.gateway.reply.v1`, key `replyTo` | Stamp `sourceTime`, `platformTime`, `freshness`, `routeMode` |
| **7** | Kafka → Gateway | — | Each instance consumes only its own partition, because the key is `replyTo` |
| **8** | Gateway → Client | HTTP | Correlate by `(correlationId, itemIndex)` · apply `projection` · assemble the ordered array · discard late replies |

### B3.4 What the gateway does in step 2

| | Action | Scope |
|---|---|---|
| a | Read the headers. In FORWARD, pass `MDS-Authentication` through unread (§B1.4) | Batch |
| b | Reject a request missing a required header — presence only, not validity | Batch |
| c | Rate limit per client; per customer once identity is known (STORAGE) | Batch |
| d | Validate batch shape and size — malformed rejects `400` **whole**, nothing dispatched | Batch |
| e | Assign `correlationId` and an absolute `deadlineAt` | Batch |
| f | Resolve the item's `serviceCode` in the cached `gateway_route` snapshot → schema refs, status, timeout. It must be a row of this path (§C2.1) | Item |
| g | Validate `args` against that service's schema | Item |
| h | STORAGE: check required offerings against the customer's entitlement. FORWARD: skipped — the provider decides | Item |
| i | The topic the deployment's mode names — the forward topic, or the row's `storage_topic` (§C2.1) | Item |
| j | Publish one envelope; a denied item completes in place and **produces no Kafka message** | Item |

> **The unit of fan-out is the batch item, not the symbol.** An item naming 200 symbols is **one** Kafka message and **one** batched store read — not 200. Splitting per symbol would break item-to-reply correlation and contradict the one-record-per-item rule. Some service calls carry no symbol at all (`marketStatus`, rankings, instrument search), which is the second reason a symbol cannot appear in a request key.

> **Why the token costs nothing here.** In FORWARD the gateway does not read the token; it rides the forward topic to the handler. The `gateway_route` snapshot is held locally. A 50-item batch costs 100 Kafka messages — two per item.

> **Why the gateway is not making policy decisions.** Route & Offering Control owns the business logic — licence state, provider capability, effective dates — and decides which services a deployment may serve. That decision reaches the gateway as the `status` of a `gateway_route` row. The gateway applies the row; it does not know *why* a service is available. It holds data, not policy.

### B3.5 Envelope

Every inter-service message on every topic uses one envelope. `serviceCode` and `args` live in the **payload**, never in the header, so the envelope never changes when a service is added.

```
header    messageId · messageType · schemaVersion · correlationId · causationId
          occurredAt · deadlineAt · replyTo · source        ( traceId optional )
security  subjectRef · scope · providerToken    ( providerToken: forward topic only )
payload   selected by messageType — opaque to the transport layer
```

Any service reads any header without knowing the payload; only the handler declaring that `messageType` deserializes it. Full definition, error envelope and error codes: **§D2**. Shipped as a shared library alongside the correlation, deadline and key-strategy helpers.

### B3.6 Topics, keys and consumer groups

| The deployment's mode | Request topic | Key | Consumer group |
|---|---|---|---|
| **`FORWARD`** — every `serviceCode` | `market.forward.request.v1` | `correlationId:itemIndex` for a batch item · the row's strategy for a single call | `mds.forward` |
| `STORAGE` — market data | `market.query.{snapshot,timeseries,reference}.request.v1` | `correlationId:itemIndex` | `mds.query.{snapshot,timeseries,reference}` |
| `STORAGE` — another business | `market.auth.request.v1` · `market.package.request.v1` | The row's strategy (§D2.6) | `mds.auth` · `mds.package` |
| Replies, in both modes | `market.gateway.reply.v1` | The partition named in `replyTo` | Static assignment, one partition per gateway instance |

**One topic per business service, not one shared topic.** The alternatives both fail:

| Alternative | Failure |
|---|---|
| One topic, one shared consumer group | Each message reaches exactly one consumer — a snapshot request could land on the time-series service |
| One topic, one group per service | Each service reads every message and discards most. Read amplification, and every service must implement filtering |

Separate topics also **structurally prevent** a query service from receiving a FORWARD message — it does not subscribe to that topic. This is enforced by topology, not by a conditional in code.

Adding a `serviceCode` needs **no new topic** — only a new business service does. A new reference-data service code joins `market.query.reference.request.v1` with one `gateway_route` row (§D1.19).

### B3.7 Example

**Request**

```json
[
  { "serviceCode": "symbolLatest",
    "args": { "exchange": "XKRX", "symbols": ["005930", "000660"] },
    "projection": ["symbol", "lastPrice", "changePercent", "volume"] },

  { "serviceCode": "symbolHistory",
    "args": { "exchange": "XHKG", "symbol": "00700",
              "resolution": "1D", "from": "2026-01-01", "to": "2026-09-01" },
    "projection": [] },

  { "serviceCode": "orderBook",
    "args": { "exchange": "XHKG", "symbol": "00700", "depth": 10 },
    "projection": ["bids", "asks"] }
]
```

**Response**

```json
[
  { "serviceCode": "symbolLatest", "isSuccess": true,
    "meta": { "routeMode": "STORAGE", "freshness": "REALTIME",
              "sourceTime": "2026-09-09T05:31:02.140Z",
              "platformTime": "2026-09-09T05:31:02.187Z" },
    "data": [ { "symbol": "005930", "lastPrice": 71200, "changePercent": 1.42, "volume": 8234100 },
              { "symbol": "000660", "lastPrice": 183500, "changePercent": -0.81, "volume": 1204400 } ] },

  { "serviceCode": "symbolHistory", "isSuccess": true,
    "meta": { "routeMode": "STORAGE", "freshness": "REALTIME", "tier": "WARM" },
    "data": { "resolution": "1D", "bars": [ ] } },

  { "serviceCode": "orderBook", "isSuccess": false,
    "error": { "code": "MARKET_DATA_ENTITLEMENT_DENIED",
               "missingOffering": "MDS:HK:ORDER_BOOK:L2:SNAPSHOT" } }
]
```

| Rule | |
|---|---|
| `args` schema | Per service, declared in the catalogue and validated at the gateway |
| `args` vocabulary | **Shared across all services** — `exchange`, `symbol`, `symbols`, `from`, `to`, `resolution`, `depth`, `limit`. A client learns the names once |
| `projection` | Top-level field names only in phase one; nested paths deferred until measured demand |
| `meta` | `routeMode` and `freshness` are what allow one DTO shape across STORAGE and FORWARD (§D4.3) |

### B3.8 The catalogue lives in the database

| Entity | Essential | Optional |
|---|---|---|
| `gateway_route` | `serviceCode`, `status`, the path it is reached through, `dispatch`, `storageTopic`, `keyStrategy`, `credentialRule`, `argsSchemaRef`, `outputSchemaRef` · `requiredOfferings` in STORAGE | display name, description, deprecation date, rate-limit class — DDL in §D1.19 |

Lifecycle `DRAFT → ACTIVE ⇄ SUSPENDED → RETIRED`, with maker-checker on publish and suspend. Published as a versioned snapshot on a compacted topic; the gateway caches it in memory, so no database read occurs per request.

> **Where the line sits.** What is **operational** — which services are live, where they route, what they require — is **data in the database**, so a service is activated or suspended without a deploy. What is **structural** — the shape of `args` and output — is **code**, because a service cannot be switched on if its handler does not exist. `argsSchemaRef` and `outputSchemaRef` point at code-resident schemas, so publishing a service whose handler is missing fails at publish time rather than at runtime.

### B3.9 Error cases

| Case | Scope | Result |
|---|---|---|
| Malformed batch — not an array, empty, over size limit, missing a required field | Whole batch | `400`, nothing dispatched |
| Required header missing | Whole batch | `401` `AUTH_TOKEN_MISSING`, nothing dispatched |
| Unknown or non-ACTIVE `serviceCode` | One item | Item error; siblings unaffected |
| `args` fail the service schema | One item | Item error |
| Entitlement denied — STORAGE | One item | `MARKET_DATA_ENTITLEMENT_DENIED` naming the missing offering; no Kafka message produced |
| Handler timeout or deadline exceeded | One item | Normalized temporary-unavailable; late reply discarded |
| Upstream provider unreachable in FORWARD | One item | `UPSTREAM_ENTITLEMENT_UNAVAILABLE` — temporarily unreachable |
| Provider rejects the token — FORWARD | One item | `AUTH_TOKEN_EXPIRED` or `AUTH_TOKEN_INVALID`, mapped from the provider's answer |
| Provider refuses the customer's rights — FORWARD | One item | `UPSTREAM_*`, never the provider's raw error |
| Kafka unavailable | Whole batch | Fails closed; MDS never bypasses the backbone |

HTTP status is `200` whenever the gateway accepted and processed the batch. Per-item outcomes live in the array — **one item failing never fails its siblings.**

### B3.10 Participating services

| Service | Role | Detail |
|---|---|---|
| Client API Gateway | Steps 2–3, 7–8 | §C2 |
| Route & Offering Control | Decides which services are available; the decision reaches the gateway as a row's `status` — **not on the request path** | §C5 |
| Query · Snapshot / Time-series / Reference | Steps 4–6, STORAGE | §C6 |
| Forward Handler | Steps 4–6, FORWARD | §C7 |

## B4. Streaming subscription — **Drafted**

### B4.1 What this section is about

![A streaming session, end to end](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-b4-session-lifecycle.png)

**Text alternative:** A client app logs in through the Client API Gateway and receives its provider session together with the partition that serves it. It opens a WebSocket to the `market-stream` instance owning that partition. Its subscribe command goes on `market.stream.command.v1` keyed by the session key; `forward-service`, which holds that customer's TTL session, picks it up and subscribes on the customer's own streaming socket. TTL pushes data back, `forward-service` fans it out on `market.stream.delivery.v1` keyed by the same session key, the owning instance consumes it, and writes the frame to that customer's socket. No other instance ever sees the session.

Streaming is the third way a client gets data, after the batch query (§B3) and the forwarded request (§B7).

| | |
|---|---|
| Transport | WebSocket. MQTT is a second adapter, for STORAGE deployments (§B4.7) |
| Entry point | `GET /api/v1/market-data/stream` — one `gateway_route` row, dispatch `STREAM` |
| Served by | `market-stream`, the Streaming Distribution service (§C10) |
| Data source | FORWARD: `forward-service`, from the customer's own provider session · STORAGE: the instrument topics |
| Scale | Kafka partitions. No cluster of sockets, no shared session store, no message bus between instances |

### B4.2 The session key

![One hash, three decisions](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-b4-session-key.png)

**Text alternative:** The provider's session id is hashed once at login, `xxHash64` over a shared salt and the session id, producing the session key; the session key modulo the partition count gives a partition `p`. That one number decides three things: the ingress routes the client's WebSocket to the instance owning `p`, Kafka places every message of this session on `p` because the session key is the message key on both streaming topics, and the `market-stream` instance owning `p` consumes it. The session id itself, like the token and the `MDS-Authentication` header, is never published.

MDS stores no provider token (§A5.3), so the session id cannot be the public identifier. It is hashed once, and only the hash travels.

| Value | Who holds it | On Kafka | In logs |
|---|---|---|---|
| `sessionID` — the provider's session | The client, and the memory of the instance holding its socket | Never | Never |
| `sessionKey` = `xxHash64(salt ‖ sessionID)` | Every service | **Yes — the message key** | Yes |
| `connectionId` | The instance holding the socket | Yes, in the delivery payload | Yes |

The salt is one shared secret from the secret manager, so every service computes the same key.

**One hash, three uses**

| Decision | How it is made |
|---|---|
| Which instance takes the WebSocket | `p = sessionKey % P`, returned by the login reply and used by the ingress |
| Which partition carries the message | `sessionKey` keys both `market.stream.command.v1` and `market.stream.delivery.v1` |
| Which instance consumes it | Static assignment: instance `i` of `M` owns every partition where `p % M == i` (§D2.7) |

Because all three come from one hash, **a session's messages always arrive at the instance holding its socket** — no routing table and no second hop. A socket that reaches the wrong instance is closed with `STREAM_WRONG_NODE` and the correct `p`; that is a misconfigured ingress, not a client error.

**A new session is a new connection.** A re-login produces a different session id, so a different hash and a different partition. Rather than carry a connection's state across that change, MDS closes the socket and the client reconnects — the path it must already have for a lost socket (§B4.4).

<mark>`xxHash64` is not a cryptographic hash. The salt must be treated as a secret: knowing it and a session id reveals the partition, though the session id cannot be recovered from the key.</mark>

<mark>Two session ids can hash alike. They then share a partition and an instance only; each socket keeps its own registry entry, so no data crosses between them.</mark>

### B4.3 Login and connect

![Login and connect](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-b4-connect.png)

**Text alternative:** The client logs in through the Client API Gateway with `auth.login` and receives the provider session and its partition. It opens a WebSocket presenting that partition; the ingress routes by it and the instance refuses a socket that is not its own. The instance hashes the session and writes one in-memory entry holding the connection id, the session, the subscriptions and the sequence per symbol. A subscribe command is published keyed by the session key; the forward-service instance owning that key subscribes upstream on the customer's own TTL session.

| # | Step | Result |
|---|---|---|
| 1 | `POST /api/v1/auth/login` | `mdsToken` — the provider session — and `streamPartition` |
| 2 | Client opens the WebSocket presenting `p` | The ingress lands it on the instance owning `p`, which verifies and issues a `connectionId` |
| 3 | Client sends `sub` | One message on `market.stream.command.v1`, key `sessionKey`, carrying the `connectionId` |
| 4 | `forward-service` subscribes upstream under the customer's own session | The provider begins pushing |
| 5 | `forward-service` publishes each update on `market.stream.delivery.v1`, key `sessionKey` | It lands on `p` |
| 6 | The owning instance finds the connection and writes the frame | One Kafka hop, one socket write |

The registry lives in memory, not in Redis: when an instance is lost its sockets are lost with it, and the client reconnects.

### B4.4 When the provider session expires

![When the provider session expires](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-b4-expiry.png)

**Text alternative:** Above, what happens when the session dies: TTL answers 401, but only to the next command MDS sends; forward-service marks that session dead and tells the socket's owner; market-stream closes the socket with `AUTH_TOKEN_EXPIRED`; the client logs in again and receives a new session and a new partition. Below, the single recovery path, the same whatever the cause — an expired session, a network fault or a deploy: log in, connect to the instance owning the new partition, subscribe again, take the snapshot and continue.

**The provider does not announce an expiry.** TTL answers `401` only to a message MDS sends it, so an expiry is learned at the next command, not at the moment it happens.

| Detection | Status |
|---|---|
| **Passive — TTL's `401` to the next command** | **In use.** The only mechanism in this design |
| Active probing on a timer, and an idle cutoff | Deferred (**D21**) |
| MDS logging the customer in again by itself | Deferred (**D22**) |

<mark>With passive detection only, a session that dies while the client is merely watching is not noticed: the stream falls silent and neither side is told until the client's next command. Closing that gap is D21.</mark>

**What happens from the moment `401` arrives**

| # | Step |
|---|---|
| 1 | `forward-service` marks that provider session dead **once**, so other items in flight for the same session do not each hit TTL with a dead id |
| 2 | It publishes one control message for that `sessionKey` |
| 3 | `market-stream` closes the socket with `AUTH_TOKEN_EXPIRED`, dropping the session's state |
| 4 | The client logs in again, receives a new session and a new `p`, connects, and subscribes again |

**One recovery path, whatever the cause.** An expiry, a network fault, a rolling deploy and a lost instance all end the same way: the socket is gone, and the client rebuilds it. That is the path a client must implement anyway, so streaming carries no second one — no grace window, no in-band token frame, and no state kept for a connection that has ended.

<mark>TTL keeps about one session per user, so an expiry is usually not a timeout but a login on another device. Two devices streaming the same user will keep ending each other's session, and each will reconnect.</mark>

### B4.5 The connection's life

![The connection's life](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-b4-lifecycle.png)

**Text alternative:** Above, the four states of a connection: OPEN once the handshake is done, the partition checked and a connection id issued; ACTIVE once the first subscribe is accepted and the upstream subscription exists; CLOSING once a reason is known and the teardown has been sent; CLOSED once the registry entry is gone. Below, what happens when it ends: market-stream closes the socket and sends `stream.close` on `market.stream.command.v1`, keyed by the session key, the topic that also carries subscribe, unsubscribe and the instance heartbeat; forward-service unsubscribes on TTL and drops the session, while the shared TTL socket stays open for other customers. Beside it, the safety net: every instance beats every five seconds, and three missed beats drop all of that instance's sessions.

A connection is state held in one instance's memory, so every way it can end must free that state — and the provider subscription it created.

| State | Entered by | Upstream |
|---|---|---|
| `OPEN` | Handshake complete, `p` verified, `connectionId` issued | Nothing yet |
| `ACTIVE` | The first `sub` accepted | `forward-service` has subscribed |
| `CLOSING` | Any reason below | Teardown sent |
| `CLOSED` | Socket closed, registry entry removed | Unsubscribed |

| Why it ends | Who notices | Close code |
|---|---|---|
| The client closes | Its close frame | Normal |
| The client vanished | Two missed pongs | `STREAM_IDLE` |
| The network dropped | The socket layer | — |
| The provider session expired | `forward-service`, at its next command (§B4.4) | `AUTH_TOKEN_EXPIRED` |
| The client cannot keep up | `market-stream` | `STREAM_SLOW_CONSUMER` |
| Deploy, or a change of instance count | `market-stream` | `STREAM_GOING_AWAY` |

**Ping and pong.** A client that loses power or is killed by its phone sends no close frame, so the socket looks alive until something asks. The server asks.

| Rule | |
|---|---|
| Mechanism | The WebSocket protocol's own ping and pong frames, not an application message — every client library answers them unprompted |
| Interval | The server pings every 15 s |
| Cutoff | Two missed pongs — about 30 s — closes with `STREAM_IDLE` |
| The client's own ping | Not required. A client that wants to measure latency uses an application frame of its own |

**The teardown.** Every ending sends one message, so the provider subscription never outlives the socket that asked for it.

```json
{ "header":  { "messageType": "ServiceRequest", "source": "market-stream/3", "…": "…" },
  "payload": { "serviceCode": "stream.close", "connectionId": "c-8f21", "reason": "STREAM_IDLE" } }
```

| Rule | |
|---|---|
| Topic and key | `market.stream.command.v1`, key `sessionKey` — the same key as the `sub` commands it undoes, so it is handled by the instance holding that upstream session, in order |
| Idempotent | Closing a session already closed does nothing |
| What `forward-service` does | Sends TTL's unsubscribe frames for that session, then drops its state. The shared socket stays open — it carries other customers |

Without this message the provider keeps pushing into a session nobody reads: wasted upstream quota, wasted Kafka traffic, and a `QUOTE_METER` the customer is still paying.

**If `market-stream` dies before it can send one.** A dead instance sends nothing, so cleanup cannot depend only on the teardown.

| Approach | Cost | |
|---|---|---|
| A heartbeat per session | ~3,300 messages/s at 100,000 sessions | Grows with customers |
| **A heartbeat per instance** — chosen | A few messages a second, whatever the session count | Does not grow with customers |

Each `market-stream` instance beats every **5 s** on the command topic. `forward-service` groups its upstream state by the `source` of the messages it received; a source silent for **three beats** has its sessions dropped. The same rule cleans up after a change of instance count.

| What | Who removes it | When |
|---|---|---|
| The in-memory registry entry | `market-stream` | The socket closes |
| The subscription on TTL | `forward-service` | `stream.close`, or the instance's heartbeat stops, or TTL answers `401` |
| The upstream session state | `forward-service` | The same moments |

### B4.6 The WebSocket protocol

| Frame | Direction | Example or content |
|---|---|---|
| `sub` | client → | `{ "op":"sub", "id":7, "channel":"quote", "exchange":"XHKG", "symbols":["00700","00005"] }` |
| `ack` | ← | `{ "op":"ack", "id":7, "ok":["00700"], "err":[{ "s":"00005", "code":"ENTITLEMENT_DENIED" }] }` — per symbol, as §B3 answers per item |
| `data` | ← | `{ "c":"quote", "s":"XHKG:00700", "t":1785378504994, "q":183, "d":{ … } }` — short names: this is the frame that repeats |
| `unsub` | client → | Same shape as `sub` |
| `end` | ← | `{ "op":"end", "s":"XHKG:00700", "code":"ENTITLEMENT_REVOKED" }` — ends one subscription, not the connection |
| `ping` · `pong` | ↔ | The protocol's own frames, every 15 s; two missed beats close the connection (§B4.5) |

`q` counts per `(connection, symbol)` and lets a client detect a gap. There is no replay and no resume: a new connection starts with a snapshot. History is the batch API's job.

**Close codes**

| Code | Meaning | What the client does |
|---|---|---|
| `AUTH_TOKEN_EXPIRED` | The provider session is dead | Log in again, then connect and subscribe |
| `STREAM_IDLE` | Two pings went unanswered | Reconnect when the client is alive again |
| `STREAM_WRONG_NODE` | The ingress routed to the wrong instance; the correct `p` is in the reason | Reconnect to `p` |
| `STREAM_SLOW_CONSUMER` | The client could not keep up | Reconnect with fewer symbols |
| `STREAM_GOING_AWAY` | The instance is draining | Reconnect |

### B4.7 A slow client must not slow the others

Conflation is the normal mode, not an emergency: a symbol ticking fifty times a second cannot be read on a screen, and a client asking for every tick pays for it in latency.

| Outbound queue for one connection | Action |
|---|---|
| Normal | Write through |
| Above the first mark | **Conflate per symbol** — quote and order book keep only the newest. Trades and bars are never conflated |
| Above the second mark | Cap at N updates per second per symbol |
| Above the third mark | Close with `STREAM_SLOW_CONSUMER` |

### B4.8 Entitlement, delayed data and transports

| Concern | FORWARD | STORAGE |
|---|---|---|
| Who allows a subscription | The provider, against the customer's own session | MDS, from the customer's entitlement, checked once at `sub` |
| Periodic revalidation | None — the provider answers `401` at the next command (§B4.4) | Server-side, per session, never touching the client (§C10.1) |
| Losing the right mid-stream | The provider stops sending; the subscription ends with an `UPSTREAM_*` code | One `end` frame for that subscription; the connection stays open |
| Realtime or delayed | Whatever the provider grants | Decided at `sub`; a delayed session is fed from the delayed consumer group and `md:dl:*` (§A9) |
| Renewal | The session changes, so the connection is rebuilt (§B4.4) | The token renews without changing whose session it is, so the connection lives (§C10.2) |

![One core, two transports](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-c10-transports.png)

**Text alternative:** The source depends on the deployment's mode: in FORWARD `market.stream.delivery.v1`, keyed by session key and produced by forward-service; in STORAGE the instrument topics, where every instance reads every partition and filters to its own sessions. Both feed one transport-neutral core holding the session registry, the subscription index, entitlement and delayed selection, conflation and back-pressure. At the edge sit two adapters: WebSocket, in use now for FORWARD, with frames, per-symbol errors and close codes; and MQTT, later, for STORAGE, publishing a topic per instrument against a broker named in the env. `STREAM_TRANSPORTS` names which adapters run.

| | WebSocket | MQTT |
|---|---|---|
| Used in | **FORWARD — today** | STORAGE — later |
| Why it fits | The stream is one customer's own provider session; nothing is shared | MDS holds every instrument, so many clients read one topic per instrument |
| Errors per symbol | In the `ack` frame | No reply channel — a per-session control topic is required |
| Entitlement | Enforced in the core; an unentitled symbol is never written | Must also be enforced as a broker ACL |

The transports are named in the env (§C10.4), never in code, and the core is identical for both.

### B4.9 Scaling and failure

| Situation | Behaviour |
|---|---|
| Adding an instance | `P` is fixed and generous (64) and is the ceiling on instance count. Changing `M` moves some partitions; the sockets on them are closed with `STREAM_GOING_AWAY` and reconnect. No long rebalance |
| Losing an instance | Its partitions pass to the survivors; its sockets were already gone, so those clients reconnect |
| Rolling deploy | Drain: stop accepting new connections, close the open ones, let clients reconnect |
| Rotating the salt | Every session hashes somewhere new, so every client reconnects. A maintenance-window operation |

### B4.10 Errors

| Condition | Result |
|---|---|
| Unknown channel, exchange or malformed frame | `ack` with `REQUEST_INVALID` |
| Symbol not entitled — STORAGE | `ack` `err` entry `ENTITLEMENT_DENIED` |
| Provider refuses the subscription — FORWARD | `ack` `err` entry `UPSTREAM_*`, never the provider's own text |
| A subscription loses its entitlement | `end` `ENTITLEMENT_REVOKED` — logging in again does not help |
| Provider session expired | Close `AUTH_TOKEN_EXPIRED` (§B4.4) |
| Client stopped answering pings | Close `STREAM_IDLE`, and the subscriptions are torn down upstream (§B4.5) |
| Wrong instance · slow client · draining | Close `STREAM_WRONG_NODE` · `STREAM_SLOW_CONSUMER` · `STREAM_GOING_AWAY` |

## B5. Delayed delivery — **Open**

Will cover: the second consumer group and its release gate, ordering guarantees across restart, and the query-time cutoff for historical data.

Mechanism already settled in §A9. The delayed stream is the same session model as §B4, fed from the delayed consumer group.

## B6. Market data ingestion (STORAGE) — **Open**

Will cover: provider feed through to normalized store — adapter emission, normalization, aggregation, and storage writing per retention policy.

Provider SPI already settled in §C8.3; the integration rule it serves is §B10.

## B7. Forward request (FORWARD) — **Drafted**

### B7.1 What this section is about

![Forward request — route and map](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-b7-forward-flow.png)

**Text alternative:** A client that has logged in to TTL MDS posts a common-API batch with its MDS token. For each item the gateway finds the `serviceCode` in `gateway_route` and publishes one message per item on the forward topic with the token as `providerToken`. The Forward Handler consumes each message, maps the platform's exchange and symbol to TTL's using its mapping configuration, builds TTL's frame with the customer's `sessID`, sends it over WebSocket and reassembles TTL's reply. It maps that reply to the service's output shape and publishes it on the reply topic, where the waiting gateway matches it by item and answers the client with one response in the order asked.

In a FORWARD deployment MDS keeps no market data. The client uses **the same common query API as every other deployment** (§B3); FORWARD only changes where each item is answered — by TTL MDS, under the customer's own session, instead of by an MDS store. MDS stands in the middle and does two things only.

| MDS does | MDS does not |
|---|---|
| **Route** — send each batch item to the provider, with the right exchange and message | Validate the token, check entitlement, store data, keep an access record |
| **Map** — the platform's exchange and symbol to the provider's, and the provider's reply to the service's output shape | Hold a provider session of its own for customer requests |

**The client's journey**

| Step | Call | Result |
|---|---|---|
| 1 | TTL core trading login — TTL's own flow, with 2FA and captcha | token 1 and the TTL MDS user mapped to the account |
| 2 | `POST /api/v1/auth/login` — §B7.2 | `mdsToken`, the TTL MDS session |
| 3 | `POST /api/v1/common/market` with `MDS-Authentication: <mdsToken>` — §B7.3 | the data |

| Service | Deployed as | Job in this flow |
|---|---|---|
| Client API Gateway | `market-api-gateway` | Labels each unit of work with its `serviceCode` and routes it; reassembles the replies |
| Forward Handler | `forward-service` | Answers every `serviceCode` in this deployment — the login of step 2 (§C1) and the query items of step 3 — and maps both ways (§C7) |

### B7.2 Step 0 — log in to TTL MDS

`POST /api/v1/auth/login` on the Client API Gateway is `serviceCode auth.login`, carried on `market.forward.request.v1` to the Forward Handler and answered on `market.gateway.reply.v1` — the messages are in §B11.6. It runs the login that ingestion already runs (`TtlSession`), for the customer's own TTL MDS user.

**Request**

```json
POST /api/v1/auth/login
Authorization: Bearer <token 1>

{ "user": "uat_qw02" }
```

| Field | | Meaning |
|---|---|---|
| `user` | Essential | The TTL MDS user mapped to the customer's core trading account, as returned by the TTL core trading login |
| `Authorization` | Optional | Passed as sent; not read |

**What `auth.login` does**

| # | Step | Detail |
|---|---|---|
| 1 | Build the login token | `hex(AES-ECB-PKCS7("{entity}:{UTC yyyyMMddHHmmssSSS}:{user}", utf8(MDS key)))` — `TtlTokenGenerator` |
| 2 | Open the TTL info socket | `websocket.info.url` of the TTL provider configuration, e.g. `wss://…/info/pubsub` |
| 3 | Send `LOGIN` | The frame below |
| 4 | Read the reply | `rtnCode = 0` → `sessID = sessionID`, or `detail.data` when `sessionID` is absent |
| 5 | Answer the client | The response below. Nothing is kept |

**`LOGIN` frame** — the shape captured in UAT

```json
{ "id": "<user>", "msgType": "LOGIN",
  "data": { "device": "<client.dataDevice>", "Agreement": "<mds.agreement>", "language": "<mds.language>",
            "entity": "<mds.entity>", "password": "<mds.password>", "token": "<login token>",
            "email": "<mds.email>" },
  "device": "<client.device>", "version": "<client.version>",
  "reqTime": 1785378494446, "entity": "<mds.entity>", "reqId": 20 }
```

`password` in this frame is the entity's shared password from configuration, not the customer's. `mds.password`, `mds.key` and `mds.email` come from the secret manager through the configuration's `passwordEnv`, `keyEnv` and `emailEnv` references; everything else is plain configuration.

**TTL's reply** — abridged

```json
{ "msgType": "LOGIN", "rtnCode": 0, "reqId": 20, "id": "uat_qw02",
  "sessionID": "<sessID>", "detail": { "id": "uat_qw02", "data": "<sessID>" },
  "service": { "HK": "Streaming", "SH": "Snapshot", "SZ": "Snapshot", "XNYS": "Streaming" } }
```

**Response to the client**

```json
{ "mdsToken": "<sessID>", "user": "uat_qw02", "streamPartition": 37,
  "services": { "XHKG": "STREAMING", "XSHG": "SNAPSHOT", "XSHE": "SNAPSHOT", "XNYS": "STREAMING" } }
```

| Field | Meaning |
|---|---|
| `mdsToken` | TTL's `sessID`, unchanged. Sent as `MDS-Authentication` on every query |
| `streamPartition` | Which `market-stream` partition serves this session — `xxHash64(salt ‖ sessID) % P` (§B4.2). The client presents it when opening a stream |
| `services` | What TTL grants this user on each exchange, wire codes mapped to platform codes (§B7.7) |

| Case | HTTP · code |
|---|---|
| `user` missing | 400 `REQUEST_INVALID` |
| TTL `rtnCode ≠ 0` | 401 `AUTH_LOGIN_REJECTED` — without TTL's own text |
| No reply within the login timeout | 504 `DOWNSTREAM_TIMEOUT` |
| TTL info socket unreachable | 503 `DOWNSTREAM_UNAVAILABLE` |

TTL keeps about one session per user: a new login ends the previous one, on any device.

### B7.3 Querying — the common API

The entry point is §B3's, unchanged: `POST /api/v1/common/market`, a batch of `{serviceCode, args, projection}`, answered in the same order. **FORWARD is not an endpoint** — it is the deployment's mode (§B11.2). In a FORWARD deployment every `serviceCode` goes to the forward topic, the market data items of this section and `auth.login` alike.

```json
POST /api/v1/common/market
MDS-Authentication: <mdsToken>

[ { "serviceCode": "symbolatest",
    "args": { "exchange": "XHKG", "symbolList": ["00700", "00005"] } },
  { "serviceCode": "chart",
    "args": { "exchange": "XHKG", "symbol": "00700", "period": "1D",
              "from": "2026-01-01T00:00:00Z", "to": "2026-09-01T00:00:00Z" },
    "projection": ["bars"] } ]
```

Every forwarded service carries `exchange` in its `args` — the shared vocabulary of §B3.7. The OpenAPI schemas gain it.

**Credentials**

| Carrier | Rule |
|---|---|
| `MDS-Authentication: <mdsToken>` | Required. Absent → `401 AUTH_TOKEN_MISSING` for the whole batch, nothing dispatched |
| `Authorization: Bearer <token 1>` | Passed as sent; not read |
| Body, when a header cannot be set | Send an object instead of the array: `{ "auth": { "mdsAuthentication": "…", "authorization": "…" }, "items": [ … ] }`. A header wins when both are present. MDS validates neither, so the carrier makes no security difference |

**Services answered by TTL**

| `serviceCode` | TTL `msgType` | TTL socket |
|---|---|---|
| `symbolatest` | `SIMPLE_PRICE_INFO` — the whole `symbolList` in one frame | snapshot |
| `orderbook` | `BEST_BID_ASK` for `depth=top` · `AGGREGATE_ORDER_BOOK` for `full` | snapshot |
| `index` | `INDEX_DATA` | snapshot |
| `chart` · `tradingview.history` · `symbolHistory` | `HISTORICAL_DATA` | snapshot |
| `company` | `SECURITY_COMPANY_INFO` | info |
| `fundamentals` | `SECURITY_FINANCIAL_DATA` | info |
| `financials/statements` | `SECURITY_FINANCIAL_REPORT` | info |
| `dividends` | `SECURITY_DIVIDEND_RECORD` | info |
| `shareholding-changes` | `SECURITY_SHARE_CHANGE` | info |

The sockets are the ones ingestion already uses: prices and history on the snapshot socket, `SECURITY_*` on the info socket. `quote`, `quoteSummary`, `marketTop`, `marketStatus`, `holiday` and `search` have no TTL message mapped yet; in a FORWARD deployment their rows stay `SUSPENDED`, so such an item returns `SERVICE_SUSPENDED` and its siblings are unaffected. Each mapping is fixed by a golden sample: a recorded TTL reply and the expected output, committed with the row.

### B7.4 Flow

| # | From → to | What the destination does |
|---|---|---|
| **1** | Client → API Gateway | Validates the batch (§B3.4); checks `MDS-Authentication` is present |
| **2** | API Gateway → forward topic | Per item: the `gateway_route` row and its `args` schema; the mode is FORWARD → `market.forward.request.v1`. One message per item (§B7.5) with the token as `providerToken` |
| **3** | Forward topic → Forward Handler | Consumes; abandons an item past `deadlineAt` |
| **4** | Forward Handler → TTL MDS | Identity mapping → TTL frame with `sessID = providerToken` → sent on the service's socket (§B7.8, §C7.4, §C7.5) |
| **5** | TTL MDS → Forward Handler | TTL checks the `sessID` and the customer's rights. The handler reassembles multi-frame replies by `reqId` until `isLast` |
| **6** | Forward Handler → reply topic | Maps TTL's reply to the service's output shape, reverses the identity, drops the token, replies with the `itemIndex` (§B7.6) |
| **7** | Reply topic → API Gateway | The instance that owns `replyTo` matches the reply by `causationId` (§C2.4) |
| **8** | API Gateway → Client | Applies `projection`, assembles the array in request order; a late reply is discarded (§B3.3) |

### B7.5 Request message — gateway to handler

One message per batch item, in the §D2 envelope. Topic `market.forward.request.v1`, key `correlationId:itemIndex`, consumer group `mds.forward`.

```
header    messageId · messageType = ServiceRequest · schemaVersion · correlationId · causationId
          occurredAt · deadlineAt · replyTo · source
security  providerToken                                   ( forward topic only · §D2.3 )
payload   serviceCode · itemIndex · args
```

| Field | Meaning |
|---|---|
| `header.correlationId` | One per HTTP batch |
| `header.deadlineAt` | Absolute; the handler abandons the item after it |
| `header.replyTo` | The waiting gateway instance's reply topic and partition (§D2.7) |
| `payload.serviceCode` | The item's `serviceCode` — the key of its `gw_operation_config` row |
| `payload.itemIndex` | The item's position in the batch |
| `payload.args` | As the client sent them; already validated |
| `security.providerToken` | `MDS-Authentication`, as received |

`projection` never leaves the gateway: the handler returns the whole output shape, and the gateway trims it before answering (§B3.3 step 8).

<mark>Forwarding a message through Kafka is itself a form of storage: the token sits on the broker's disk until retention removes it.</mark>

So the forward topic keeps a message for as short as it can — `retention.ms = 60000`, well above the longest item timeout — and `providerToken` is masked in every log line, trace attribute and dead-letter record.

### B7.6 Reply message — handler to gateway

Topic `market.gateway.reply.v1`, to the partition named in `replyTo`. Same envelope; `causationId` is the request's `messageId`.

```
payload   serviceCode · itemIndex · isSuccess · data · error { code, message, retryable }
          meta { routeMode, freshness, sourceTime, platformTime }
```

| Field | Rule |
|---|---|
| `data` | The service's output shape in the OpenAPI contract. What TTL supplies is mapped; what it does not supply is `null` |
| `meta.routeMode` | `FORWARD` |
| `meta.freshness` | `REALTIME` or `DELAYED`, from the TTL message used |
| `meta.sourceTime` | TTL's `msgTime` |
| `error` | A code from §B7.9. Never TTL's own text |

### B7.7 Identity mapping

| Direction | Exchange | Symbol |
|---|---|---|
| Outbound | `gw_exchange_map`: `XHKG → HK`, `XSHG → SH`, `XSHE → SZ`, `XNYS → XNYS` | `gw_symbol_rule`: `STRIP_ZERO` for HK (`00700 → 700`), `PAD6` for SH and SZ, `PASSTHROUGH` for US and indices |
| Inbound | The reverse — also applied to the login reply's `service` map | The reverse (`700 → 00700`) |

An unmapped exchange or symbol is rejected before any TTL call. The rules are a closed set in code; the tables only choose which rule applies.

### B7.8 The TTL frame

Every query frame carries the same fields ingestion sends today:

```json
{ "msgType": "SECURITY_FINANCIAL_DATA", "symbol": { "exchangeID": "HK", "key": "700" },
  "language": "en", "subscribe": null,
  "device": "<client.device>", "version": "<client.version>", "reqTime": 1785378504994,
  "entity": "<mds.entity>", "reqId": 21, "sessID": "<providerToken>" }
```

Price requests carry `symbols: [{exchangeID, key}, …]` and `msgTypes: [...]` instead of `symbol` and `msgType`, with `subscribe: null` for a one-off snapshot. **The only per-customer value in any frame is `sessID`**; everything else is the deployment's TTL configuration.

### B7.9 Errors

| Condition | Scope | Code | HTTP |
|---|---|---|---|
| `MDS-Authentication` absent | Batch | `AUTH_TOKEN_MISSING` | 401 |
| Malformed batch | Batch | `REQUEST_INVALID` | 400 |
| Service unknown, not reachable through this path, or `SUSPENDED` | Item | `SERVICE_NOT_FOUND` · `REQUEST_SERVICE_NOT_ALLOWED` · `SERVICE_SUSPENDED` | 200, item error |
| `args` fail the schema | Item | `INVALID_ARGS` | 200, item error |
| Exchange or symbol unmapped | Item | `INSTRUMENT_NOT_FOUND` | 200, item error |
| TTL `rtnCode 401` — session expired or invalid | Item | `AUTH_TOKEN_EXPIRED` — the client logs in again (§B7.2) | 200, item error |
| TTL `rtnCode` other non-zero | Item | `DOWNSTREAM_ERROR` | 200, item error |
| TTL returns no data | Item | `DATA_NOT_FOUND` | 200, item error |
| TTL quota exhausted (`QUOTE_METER remain = 0`) · pool saturated | Item | `DOWNSTREAM_BUSY` | 200, item error |
| TTL unreachable | Item | `DOWNSTREAM_UNAVAILABLE` | 200, item error |
| No TTL reply within the row's `timeout_ms`, or none before `deadlineAt` | Item | `DOWNSTREAM_TIMEOUT` | 200, item error |

As §B3.9 sets out: HTTP is `200` whenever the batch was accepted, and one item failing never fails its siblings. The handler never logs in again on a `401` — the session is the customer's. `DOWNSTREAM_*` is today's name for §D2.10's `UPSTREAM_*`, renamed with the §D2.14 migration.

### B7.10 Configuration to seed

| What | Where | Content for the HK / US FORWARD deployment |
|---|---|---|
| Mode and fixed topics | The gateway's env | `GATEWAY_MODE=FORWARD` · `GATEWAY_FORWARD_TOPIC=market.forward.request.v1` · `GATEWAY_REPLY_TOPIC=market.gateway.reply.v1` · `GATEWAY_REPLY_PARTITION` per pod |
| Routes | `mds.gateway_route` | One row per service in §B7.3, plus `auth.login`. Services with no TTL message: `SUSPENDED` |
| Exchange map · symbol rule | `mds.gw_exchange_map` · `mds.gw_symbol_rule` | The rows in §B7.7 |
| Provider mapping | `mds.gw_operation_config` | One row per `serviceCode`, including `auth.login` (§C1.2) |
| Enablement | `mds.gw_enablement` | On for the deployment's exchanges |
| TTL provider configuration | provider snapshot | The descriptor ingestion already uses — `mds.*`, `client.*`, `websocket.info` and `websocket.snapshot` URLs; secrets by reference |
| Topic retention | `market.forward.request.v1` · `market.gateway.reply.v1` | `retention.ms = 60000` |

Every table's DDL and the seed rows are in §D1.19. A service fails fast at startup if one of its tables is missing.

### B7.11 Outside this section

| Topic | Where |
|---|---|
| Why two tokens, and who issues what | §A5, §B1 |
| How the gateway turns a path into a `serviceCode` and a topic | §C2 |
| Streaming — subscribe and push | §B4 |
| STORAGE deployments | §B3, §C6 |

## B8. Offering and package administration — **Open**

Will cover: catalogue lifecycle, maker-checker approval, versioned publication, and the effect of suspending an offering on live subscriptions.

## B9. Licence reporting — **Drafted**

> Applies to STORAGE deployments. A FORWARD deployment keeps no access record: each call reaches the provider under the customer's own token, so the provider holds the per-customer record.

### B9.1 What is produced

Each exchange requires a periodic declaration of how many entitled users the distributor served, split by user class and by data product. The **declaration** is the contractual artefact, not the audit trail behind it, and it must still be defensible several years after it was filed.

**`token_decision_audit` is deliberately not the counting source.** It is a hashed, per-decision forensic record kept to answer "why was this specific request allowed or refused" in a dispute. Counting from it would be counting request decisions, not entitled users. The counting sources are subscription state and daily access.

### B9.2 Sources

![Producing a monthly declaration](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-b9-licence-declaration.png)

**Text alternative:** Subscription state, the daily access summary and the customer's user class feed an offline declaration job. The job runs after the period closes, reads its sources up to a recorded watermark, and writes a declaration header with one line per user class and offering. The declaration passes maker-checker approval before it is exported for submission, and is never edited afterwards.

| Layer | Entity | Why it can be used as evidence |
|---|---|---|
| Subscription state | `customer_subscription` + `subscription_event` (§D1.14) | append-only; the event log is what a claim about a past period is defended with |
| Access | `daily_access_summary` (§D1.15) | written once per customer per exchange per day, and for FORWARD markets it is the **only** per-customer record that exists (§D5) |
| Classification | `customer.user_class` (§D1.13) | exchange declarations count and price `RETAIL` and `PROFESSIONAL` separately |

### B9.3 The counting basis — **Open (D16)**

The one decision that changes the number. Exchange contracts differ, so this is answered per exchange, not once.

| Basis | Counts | Consequence |
|---|---|---|
| **Entitlement** | every customer whose subscription granted that exchange at any point in the period | highest count; easiest to defend because it matches what was sold |
| **Access** | customers with at least one `daily_access_summary` row for that exchange in the period | lower; depends on the access record being complete, which for FORWARD markets it is by construction |
| **Period-end snapshot** | customers entitled on the last day of the period | lowest and most volatile — a customer who cancelled mid-period disappears entirely |

**Suggested default:** the entitlement basis, with the access basis available per exchange where a contract asks for it. Under-declaring is a licence breach; over-declaring is a cost. The entitlement basis errs toward the recoverable mistake.

**Design consequence either way:** the basis is stored **on the declaration**, not held in the job's code. A change of basis then appears in the record rather than silently changing next month's number.

### B9.4 Declaration entities — **New**

| Entity | Essential | Optional |
|---|---|---|
| `licence_declaration` | exchange, period, state (`GENERATED` / `APPROVED` / `SUBMITTED` / `SUPERSEDED`), counting basis, source watermark, generated at | approved by, submitted at, supersedes reference, note |
| `licence_declaration_line` | declaration, user class, offering, count | breakdown reference |

| Rule | Reason |
|---|---|
| A submitted declaration is never edited | it is the artefact an exchange audit is answered with; a mutated number is indefensible |
| A correction is a **new** declaration superseding the previous one; both are retained | the correction history is itself evidence of good faith |
| The job records the watermark it read to | re-running it years later must reproduce the same number, not today's number |
| Every line is traceable to the source rows that produced it | "where did this figure come from" is the first question in an audit |

### B9.5 The job

| Property | Design |
|---|---|
| Trigger | after the period closes, plus a settlement lag configurable per exchange |
| Placement | admin and reporting service, offline. It never touches the request path |
| Determinism | reads only append-only or day-closed sources, up to a recorded watermark |
| Re-run | idempotent: produces a `GENERATED` result and refuses to overwrite one already approved |
| Late data | source rows arriving after the watermark are picked up by a restatement, never by silently changing a filed figure |

### B9.6 Approval and output

Maker-checker, the same model as catalogue administration (§B8): the job generates, a reviewer approves, and only an approved declaration can be exported. The export format and submission channel are per-exchange and are business input — **Open (D17)**.

### B9.7 Not yet designed

| Item | Needed from |
|---|---|
| Per-exchange declaration format and submission channel | BA / exchange contracts — **D17** |
| Whether any exchange requires a period shorter than monthly, or a breakdown finer than user class and offering | exchange contracts |
| Whether a declaration must be retained beyond the seven-year audit period assumed in §D1.15 | Legal |

---

## B10. Provider integration and the adapter layer — **Drafted**

### B10.1 What this section is about

![Where the adapter layer sits](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-b10-layer-context.png)

**Text alternative:** The same layers as the architecture figure in §A3, shown as two paths. Along the top, a client application sends a request to the API entry layer, which passes it to the platform business layer. Along the bottom, market data providers send raw data into the provider integration layer, which translates it into platform form and stores it; the business layer reads that stored data. The business layer never speaks to a provider.

The platform is built in layers, each with one job. This section is about **one of them** — the layer highlighted above.

| Layer | Its job, in plain terms | Changes when |
|---|---|---|
| API entry | Take a request, work out who is asking and what they are allowed to see | the public contract changes |
| Platform business | The rules of the product: what a query returns, what a subscription grants, what is charged | the product changes |
| Storage | Hold market data in the platform's own form | the data model changes |
| **Provider integration** | **Connect to each provider, read what it sends, and translate it into the platform's own form before it goes anywhere else** | **a provider is added, replaced or changed** |

**What the provider integration layer is for.** A provider — TTL, ICE, whoever comes next — has its own way of connecting, its own codes for exchanges, its own symbols, its own field names, its own idea of what a price message looks like. None of that is the platform's business. This layer absorbs all of it and hands the rest of the platform one consistent shape.

**Why that matters commercially.** The rules of the product are the part that took the longest to get right and the part a customer depends on. Confining every provider difference to one layer is what lets those rules stay still. Adding the fifth market should cost roughly what adding the second did, and should not put the first four at risk. That is the whole design goal of this section; everything below is how it is achieved and how it is enforced.

### B10.2 Vocabulary used in this section

Four terms appear throughout. They are ordinary software terms, but the design does not make sense if they are read loosely.

| Term | What it means here | Example in this system |
|---|---|---|
| **Package** | One folder of code that ships as a unit. Code inside a package can be rewritten freely; code outside it should not need to notice | `adapter/provider/ttl` is the TTL package. Everything TTL-specific lives in it |
| **Port** | A written list of what the platform needs done, and nothing about who does it or how. It is a contract, not code that runs | `MarketDataProvider` says "subscribe to this exchange". It does not say socket, or login, or message format |
| **Adapter** | The code that fulfils a port for **one** specific outside system | The TTL adapter fulfils "subscribe to this exchange" by opening a TTL socket and sending TTL's own login |
| **Platform form** (canonical form) | The platform's own vocabulary for a record: exchange as a MIC code, symbol as the platform's symbol, time in UTC | A price from any provider arrives downstream in exactly this shape — see §B10.8 |

**The arrangement has a name.** Declaring ports in the core and putting every outside system behind an adapter is called *ports and adapters*, or *hexagonal architecture*. It is a standard, long-established pattern rather than something invented for this platform. Knowing the name is not needed to read on; it is given so that the approach can be checked against the literature, and so that a new developer recognises what they are looking at.

### B10.3 The question a customer asks

> *How is the platform designed so that integrating a new provider does not require changing the platform's business code?*

The goal was stated in §B10.1. What follows is how it is made checkable, because "no business change" as a claim is worth nothing unless it can be tested. It is four separate claims:

| # | Claim | Checked by |
|---|---|---|
| 1 | No file outside one adapter package names the provider | build-enforced dependency test (§B10.11) |
| 2 | How a record is treated downstream never depends on which provider produced it | golden files per provider, asserted against the same output shape |
| 3 | Adding a provider cannot change an existing provider's output | every existing provider's golden files re-run in CI |
| 4 | Changing which provider serves an exchange is configuration, not a release | route published as a configuration snapshot (§B10.7) |

### B10.4 The rule

One rule produces all four, and it is a rule about **which code is allowed to know about which**.

The platform's own code — the canonical model, the ports, and the services that use them — is called `core` below. It is written first, and it knows nothing about any provider. Every adapter is written against `core`. Nothing is ever written the other way round: `core` never reaches out to an adapter, and cannot even see that one exists.

| Package | May depend on | May never depend on |
|---|---|---|
| `core` — model, ports, services | the JDK and its own ports | any adapter, any provider SDK, any vendor-binding annotation |
| `adapter/provider/<code>` | `core`, that provider's own SDK | another provider adapter |
| `adapter/publisher` | `core`, the Kafka client | any provider adapter |
| `infra` — configuration, health | `core`, Spring, Kafka | any provider adapter |
| `api` — operational surface | `core` | any provider adapter |

The consequence is the whole answer: **because `core` cannot see an adapter, no adapter can be added that changes it.**

### B10.5 The ports

![Ports and adapters](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-b10-adapter-layer.png)

**Text alternative:** Four columns. The platform core holds the registry, the canonical model and configuration state, and names no provider. The ports column declares the provider, translation, output and configuration interfaces. The adapters column implements them — one package per provider, plus the Kafka publisher and the configuration listener. The right column holds the external systems each adapter speaks to. Arrows show dependency: the core and the adapters both point at the ports, and neither points at the other.

| Port family | Interfaces | Implemented by | What provider N+1 does to it |
|---|---|---|---|
| Provider | `AdapterManagerFactory`, `AdapterManager`, `MarketDataProvider`, `CrawlProvider`, `ReferenceProvider`, `ProviderSession` | the new adapter package | one new implementation; the interfaces are untouched |
| Translation | `CanonicalExchangeResolver`, `ExchangeWireCodeResolver` | configuration | new rows, no code |
| Output | `EnvelopeSink`, `MarketDataPublisher` | the publisher adapter | nothing |
| Configuration | `ProviderConfigPort`, `MarketExchangeConfigPort`, `MarketSymbolConfigPort`, `SymbolUniversePort`, `HistoricalSymbolConfigPort` | the snapshot listener | nothing |

Method-by-method function is in §C8.3.

**Selection is not a switch statement.** `ProviderFactory` holds a map from provider code to `AdapterManagerFactory`, built from every factory on the classpath. A new adapter registers itself by existing; the dispatcher is never edited. An unknown provider code is a typed failure, not a silent no-op.

### B10.6 Capability negotiation

Providers differ in what they can serve. A single fat interface would force every adapter to implement methods it cannot honour and would push "not supported" into runtime exceptions. Capability is therefore **declared**, not discovered by failure.

| Declared by | Value | Effect when absent |
|---|---|---|
| `AdapterManager.streaming()` | optional `MarketDataProvider` | no streaming session is opened and no subscribe work is scheduled |
| `AdapterManager.crawl()` | optional `CrawlProvider` | the crawl scheduler never dispatches to this provider |
| `AdapterManager.reference()` | optional `ReferenceProvider` | reference refresh skips it |
| `CrawlProvider.supportedKinds()` | `HISTORICAL` / `FINANCIAL` | only servable kinds are dispatched |

A provider that only crawls, or only streams, is a normal case rather than an error case — and no caller contains a provider-specific branch.

### B10.7 Configuration or code

The practical answer to the customer's question. For a new provider:

| Concern | Where it lives | Code change |
|---|---|---|
| Credentials | secret manager; the configuration carries only a reference | No |
| Endpoint, ports, timeouts, protocol options | `provider_config.settings` — an opaque blob the core never interprets | No |
| Which exchanges this provider serves | `provider_exchange` | No |
| Exchange code ↔ MIC, in both directions | `provider_exchange` wire codes | No |
| Provider symbol ↔ canonical symbol | `instrument_provider_mapping` | No |
| Which provider serves an exchange in a region | `region_exchange_access` | No |
| The symbol universe to subscribe | symbol snapshot | No |
| Enabling or disabling a market at runtime | registry command | No |
| **Wire protocol — framing, parsing, session, login** | a new adapter package | **Yes — one package, isolated** |
| **A field the canonical model has no place for** | canonical model and storage | **Yes — §B10.9** |

Only the last two rows are code, and only the last one reaches beyond the adapter package.

### B10.8 Where the provider stops existing

![Where the provider stops existing](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-b10-event-path.png)

**Text alternative:** Streaming frames, crawl responses and reference responses each enter their own adapter, which parses the provider's wire format and resolves the provider's exchange code and symbol into canonical form. All three converge on one MarketEnvelope carrying no provider vocabulary, which leaves through EnvelopeSink to a topic the core chooses.

| Rule | Reason |
|---|---|
| Exchange and symbol resolve to canonical form **before** the envelope is constructed | a provider code must never be able to reach storage or a client contract |
| An unmapped exchange or symbol raises a typed error | forwarding an unmapped code silently corrupts data that is expensive to find later |
| Timestamps are UTC; provider event time and platform ingest time are both carried | lag is measurable without a second source |
| The adapter never chooses a topic or a key | topic layout is a platform decision (§D2); an adapter choosing one couples routing to a vendor |
| The adapter never publishes | its only output is `EnvelopeSink` |

### B10.9 The one seam that does cost

A provider carrying a field the canonical model has no place for is the only case that reaches past the adapter. Three answers, in order of preference:

| Option | Use when | Cost |
|---|---|---|
| Drop the field | it appears in no client contract and no licence obligation | none |
| Add a nullable field to the canonical record | the field is meaningful across providers, not only this one | one additive model change; existing adapters are unchanged because the field is nullable; `schemaVersion` increments and consumers tolerate unknown fields |
| Store it as an instrument attribute (§D1.4) | the field is provider- or market-specific | no model change; reachable through the attribute path |

**The rule that keeps this from eroding:** never add a provider-named field to a canonical record. If a field cannot be named without the provider's vocabulary, it belongs in the attribute model, not in the canonical record.

### B10.10 Onboarding a new provider

| # | Step | Where | Produces code |
|---|---|---|---|
| 1 | Register the provider and its secret references | configuration | No |
| 2 | Declare its exchanges and wire codes in both directions | configuration | No |
| 3 | Load the symbol mapping for the instruments in scope | configuration | No |
| 4 | Write the adapter: factory, manager, session, parsers | `adapter/provider/<code>` | **one new package** |
| 5 | Declare which capabilities it serves | in that manager | in that package |
| 6 | Record golden files — captured upstream payload to expected envelope | that package's test resources | in that package |
| 7 | Run the conformance suite (§C8.12) | CI | No |
| 8 | Point an exchange at the provider and publish the snapshot | configuration | No |

Steps 4–6 are the only ones that produce code, and all of it lands under one directory.

### B10.11 How the rule is held

| Guard | What it catches | Status |
|---|---|---|
| Dependency test in CI — no class in `core` may import an adapter package, and no identifier in `core` may contain a provider code | the leak that makes provider N+2 expensive | **To build.** The current code has five violations (§B10.12) |
| Golden files per provider, re-run on every build | a new provider silently changing an existing provider's output | with the conformance suite (§C8.12) |
| Conformance suite | an adapter that compiles but breaks a cross-provider invariant | delivered with the ICE adapter |

The dependency test is the load-bearing one. The rule in §B10.4 is only worth stating if a build can fail on it; without that it degrades on the first deadline.

### B10.12 Known deviations in the current code — **to fix**

Recorded because they are the exact places where the second provider's work would land in a file named after the first.

| # | What | Where | Why it breaks the rule | Fix |
|---|---|---|---|---|
| 1 | `TtlTokenGenerator` | `core/service` | a provider's login crypto sitting in the core | move it into `adapter/provider/ttl`; it is used only from there |
| 2 | `TtlHistoricalCrawlException` | `core/exception` | a core type named after a provider | generalise to a crawl-failure type with a reason enum |
| 3 | Canonical records documented in terms of one provider's message names | `core/model/standardized` | the model may have been shaped by that provider rather than by the domain | review each record against the ICE adapter — the second provider is the test |
| 4 | Provider-named operator command types and listener | `infra/config/admin`, `infra/config/snapshot` | the operator command path is provider-specific | a provider-neutral command envelope carrying a provider-specific payload |
| 5 | Secret resolution knows one provider's config shape — it resolves a fixed set of key names | `infra/config/snapshot` | adding a provider with different secret fields requires editing shared configuration code | resolve by convention, or from a descriptor the adapter itself declares |

None of these break anything today. **#3 is the one to settle first** — it is a design question rather than a rename, and the ICE adapter is what answers it.


## B11. Service communication and topics — **Drafted**

### B11.1 What this section is about

![One gateway, two modes](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/gwd-01-two-modes.png)

**Text alternative:** A client app calls the Client API Gateway, which reads its mode from its env file and its routes from the `gateway_route` table. In FORWARD every serviceCode — login, chart, packages alike — goes on `market.forward.request.v1` to the Forward Handler, which decides what each one means and maps it to the provider's API, reaching TTL outside MDS. In STORAGE each serviceCode goes to the topic of its business: market data queries to the query topics, `auth.login` to `market.auth.request.v1`, the package calls to `market.package.request.v1`; each of those services decides for itself whether to read MDS storage, its own database or the trading core. Every reply returns on `market.gateway.reply.v1`.

MDS services talk to each other only through Kafka (§A1). **Each layer knows only what it needs to**, and that is what decides the topics.

| Layer | Knows | Does not know |
|---|---|---|
| **Client API Gateway** | The deployment's mode, each `serviceCode`, and the topic that `serviceCode` goes to — an address it never interprets | What a `serviceCode` means, which provider answers it, where the data comes from |
| **Forward Handler** | Every `serviceCode` in a FORWARD deployment: which provider message, which socket, how to map the reply | Who the client is, what the HTTP call looked like |
| **Business services** — market data query, auth, package | The `serviceCode`s it owns, and how to answer each: its own store, its own database, or a call to the trading core | The HTTP surface, the client's transport |

**`serviceCode` is the only name for what to do.** A client sends it in the common query API; for any other endpoint the gateway labels the request from its configuration. It travels in the message and the receiving service dispatches on it (§D2.2).

### B11.2 The mode decides the topic — **Drafted**

| Mode | Request topic | Handled by | Decided by |
|---|---|---|---|
| **`FORWARD`** | **`market.forward.request.v1`** — every `serviceCode` | Forward Handler — `forward-service` | `GATEWAY_MODE` in the gateway's env |
| `STORAGE` | The `serviceCode`'s `storage_topic` — one per business | The business service that owns it | The same env variable, then the row |

A deployment runs one mode (§A8), so the mode is a deployment setting, never a per-request decision. The same rows serve both: in FORWARD the `storage_topic` column is simply not read.

**FORWARD and STORAGE are not the client's business.** Neither the client nor the gateway knows how an answer is produced. In STORAGE the package service may hold the subscription itself or proxy the call to the trading core; that choice is the service's alone and changes nothing above it.

### B11.3 The splitting rule — **Drafted**

| Option | What happens |
|---|---|
| One topic for everything | Every service reads every message and discards most; one retention for tokens and market data alike |
| One topic per endpoint or per API | Every new API needs a topic, consumer wiring and a release |
| **One topic per business service** — chosen | FORWARD is one service, so one topic. In STORAGE each business owns one. A new API is one row |

A new topic is justified only when one of these differs; an API, a `serviceCode` or a provider never creates one.

| # | Test | Example |
|---|---|---|
| 1 | A different service handles it | FORWARD: the Forward Handler. STORAGE: market data query, auth, package |
| 2 | Retention or security differs | `market.forward.request.v1` carries sessions, so it keeps one minute |
| 3 | The kind of conversation differs (§B11.5) | A snapshot must be compacted; a request must not be |

**A request topic is named after the service that owns it, never after a verb.** `market.query.*` is right for market data reads and wrong for a package subscription, which is a write; the owner's name stays true for both. Names follow §D2.4.

**The key says what must stay in order** (§D2.6). Market data items need no order and spread by `correlationId:itemIndex`; a login keys on the user; a subscription keys on the customer, so that customer's subscribe and cancel arrive in order on one partition.

### B11.4 Replies follow the service that waits

Every reply for the Client API Gateway — FORWARD or STORAGE — goes to `market.gateway.reply.v1`. Each gateway instance owns one partition and names it in `replyTo` (§D2.7); a reply finds its request by `causationId` (§C2.4). The topic keeps one minute, because a login reply carries a session id; nothing ever replays a reply.

### B11.5 Between services — never through the API gateway

| Kind | Shape | Who waits |
|---|---|---|
| Request / reply | One request, one reply, a deadline | The client, through the gateway — §B11.2 |
| Command / outcome | An instruction and its result | An operator |
| Event | A fact; any number of readers; ordered by key | Nobody |
| Snapshot | The latest state of a thing; compacted | Nobody — read at startup and on change |

| Flow | Path | Topic | Key | Retention |
|---|---|---|---|---|
| Market data feed — STORAGE only | Ingestion, from the provider → Processing → storage writers → database; streaming reads the same events | `market.instrument.{trade,quote,bar,reference,financial,universe}.v1` | `exchange:symbol` | Replay window |
| Configuration | admin-service → every instance that needs it | `market.config.{gateway-route,forward-mapping,provider,offering}.v1` | Subject id | Compacted |
| Admin change | Admin API Gateway → admin services | `market.admin.command.v1` · `market.admin.outcome.v1` | `entityRef` | Audit period |
| TTL operator action — crawl, interactive login | market-admin → Ingestion | `market.admin.ttl.command.v1` · `market.admin.ttl.reply.v1` | — | 1 hour |
| Subscription changed | Package service → Client API Gateway | `market.subscription.changed.v1` | `subjectRef` | Audit period |
| Charge | Package service → Trading core | `market.billing.charge.request.v1` · `.outcome.v1` | `subscriptionRef` | Audit period |

**Query and the market data feed are two conversations, not two copies of one.**

| | Market data feed | Common query |
|---|---|---|
| Direction | **Write** — the provider pushes, MDS stores | **Read** — a client asks, MDS answers |
| Path | Ingestion → Processing → database. **Never the API gateway** | Client → API gateway → service → API gateway → client |
| Who waits | Nobody | The client |
| Key | `exchange:symbol` — one instrument's events stay in order | `correlationId:itemIndex` — no order needed |
| Kept | The replay window, so derived data can be rebuilt | Minutes at most |
| In FORWARD | **Does not exist** — nothing is stored | Answered by the provider (§B7) |

### B11.6 Login, end to end — the messages

| # | From → to | Carried on | What the destination does |
|---|---|---|---|
| **1** | Client → Client API Gateway | HTTPS `POST /api/v1/auth/login` | The `gateway_route` row for that path gives `serviceCode auth.login`, `dispatch SINGLE`, key `ARG(user)`. Validates the body against the row's `args` schema |
| **2** | Gateway → Forward Handler | `market.forward.request.v1` in FORWARD — `market.auth.request.v1` in STORAGE — key the user, group `mds.forward` | Finds its `gw_operation_config` row for `auth.login`: `LOGIN` on the info socket, no session needed |
| **3** | Forward Handler → TTL MDS | TTL info socket | Builds and sends TTL's `LOGIN` frame exactly as ingestion does (§B7.2) |
| **4** | Forward Handler → Gateway | `market.gateway.reply.v1`, to the partition in `replyTo` | The reply, mapped by the row's `response_jsonata` |
| **5** | Gateway → Client | HTTPS | `200` with `data`, or the HTTP status of the error (§B7.2) |

**Request** on `market.forward.request.v1`

```json
{ "header":  { "messageId": "9f2c…", "messageType": "ServiceRequest", "schemaVersion": 1,
               "correlationId": "5b1e…", "causationId": "5b1e…",
               "occurredAt": "2026-09-11T03:10:00Z", "deadlineAt": "2026-09-11T03:10:10Z",
               "replyTo": { "topic": "market.gateway.reply.v1", "partition": 3 },
               "source": "market-api-gateway/2" },
  "payload": { "serviceCode": "auth.login", "requestId": "a71e…", "args": { "user": "uat_qw02" } } }
```

**Reply** on `market.gateway.reply.v1`

```json
{ "header":  { "messageId": "c41a…", "messageType": "ServiceReply", "schemaVersion": 1,
               "correlationId": "5b1e…", "causationId": "9f2c…",
               "occurredAt": "2026-09-11T03:10:00.412Z", "source": "forward-service/1" },
  "payload": { "serviceCode": "auth.login", "isSuccess": true,
               "data": { "mdsToken": "<sessID>", "user": "uat_qw02", "streamPartition": 37,
                         "services": { "XHKG": "STREAMING", "XNYS": "STREAMING" } } } }
```

**A query item on the same topic**, for comparison — only the payload and the security block differ:

```json
{ "header":   { "messageType": "ServiceRequest", "correlationId": "77d0…", "…": "…" },
  "security": { "providerToken": "<mdsToken>" },
  "payload":  { "serviceCode": "chart", "itemIndex": 1,
                "args": { "exchange": "XHKG", "symbol": "00700", "period": "1D",
                          "from": "2026-01-01T00:00:00Z", "to": "2026-09-01T00:00:00Z" } } }
```

Neither message names a provider: the gateway does not know one. The Forward Handler serves the provider its deployment is configured for.

<mark>Forwarding a message through Kafka is itself a form of storage.</mark> The login reply carries a session id and every query item carries one, so `market.forward.request.v1` and `market.gateway.reply.v1` keep one minute, and the value is masked in every log and trace.

### B11.7 Adding an API — worked examples

| Case | At the gateway | In the handling service | Topic |
|---|---|---|---|
| **A** · a new market data read | One `gateway_route` row on the common path | FORWARD: a `gw_operation_config` row. STORAGE: the query service's code | None |
| **B** · a business with its own endpoints — the package calls | One `gateway_route` row per endpoint | The package service's code | None once `market.package.request.v1` exists |
| **C** · a new business service | The rows of its endpoints | The new service | One: `market.<business>.request.v1` |

**Case B in full** — three endpoints, three rows, no new topic and no gateway change

```sql
INSERT INTO mds.gateway_route
  (service_code, http_method, path, dispatch, storage_topic, key_strategy,
   credential_rule, args_schema_ref, timeout_ms, status) VALUES
  ('package.list',      'GET',  '/api/v1/packages',           'SINGLE', 'market.package.request.v1', 'REQUEST_ID', 'MDS_AUTH_REQUIRED', 'PackageListArgs',      5000,  'ACTIVE'),
  ('package.subscribe', 'POST', '/api/v1/packages/subscribe', 'SINGLE', 'market.package.request.v1', 'SUBJECT',    'MDS_AUTH_REQUIRED', 'PackageSubscribeArgs', 10000, 'ACTIVE'),
  ('package.cancel',    'POST', '/api/v1/packages/cancel',    'SINGLE', 'market.package.request.v1', 'SUBJECT',    'MDS_AUTH_REQUIRED', 'PackageCancelArgs',    10000, 'ACTIVE');
```

**What the package service receives**, keyed by the customer so one customer's calls stay in order:

```json
{ "header":   { "messageType": "ServiceRequest", "correlationId": "e20b…",
                "replyTo": { "topic": "market.gateway.reply.v1", "partition": 1 }, "…": "…" },
  "security": { "subjectRef": "<hashed customer>" },
  "payload":  { "serviceCode": "package.subscribe", "requestId": "<Idempotency-Key>",
                "args": { "packageCode": "HK-L2" } } }
```

It replies once on `market.gateway.reply.v1`, and publishes `market.subscription.changed.v1` for everyone else (§B11.5). A second message with the same `requestId` returns the first outcome. Whether it wrote its own database or called the trading core is invisible above it.

| Needs code | Needs only rows |
|---|---|
| A new dispatch mode or key strategy (§C2.2) | A new endpoint of an existing mode |
| A new JSONata helper function (§C7.6) | A new FORWARD `serviceCode` |
| A new business service, with its topic | A new `serviceCode` of an existing service |

### B11.8 Placing a new API

| Question | Answer |
|---|---|
| Is it a market data read a client batches? | Yes → a row on `POST /api/v1/common/market`. No → its own `SINGLE` endpoint |
| Does it change state? | Yes → key `SUBJECT` or `REQUEST_ID` with an `Idempotency-Key`, and never a batch item |
| Who answers it? | In FORWARD always the Forward Handler. In STORAGE the business service named by `storage_topic` |
| How is the answer produced? | The handling service's decision — its store, its database, or the trading core. Never the gateway's concern |
| Is it a fact others react to, or latest-state configuration? | Not a client API — an event topic or a compacted config topic (§B11.5) |

---

# Part C — Service Design

## C1. Login relay — a Forward Handler service code — **Drafted**

### C1.1 Placement — **Settled**

Login is not a separate service in a FORWARD deployment. To the gateway it is `serviceCode auth.login`, routed exactly like `chart`; the Forward Handler (§C7) answers it, because it needs what the Forward Handler already owns: the provider's protocol, its info socket and its configuration. It exists for the interim, while the Login Server is not live (D18). In a STORAGE deployment the same `serviceCode` is answered by the auth service on `market.auth.request.v1`, and nothing above changes. API contract in §B7.2; messages in §B11.6.

### C1.2 What `auth.login` does

| Does | Does not |
|---|---|
| Receive `{ serviceCode: "auth.login", args: { user } }` and reply on `market.gateway.reply.v1` | Serve HTTP — the Client API Gateway owns `POST /api/v1/auth/login` |
| Build the login token and the `LOGIN` frame exactly as ingestion's `TtlSession` does, from the TTL provider configuration | Ask for the customer's password or a second factor — the TTL MDS login needs neither |
| Send it on the TTL info socket and read `sessID` from the reply | Issue, sign, wrap or refresh a token of its own |
| Return `sessID` unchanged, with the `service` map in platform codes | Store the session id, the key, or a session |
| Report a failed login without TTL's own error text | Log a request body · resolve entitlement |

It is one `gw_operation_config` row — `(auth.login, TTL)`, `msg_type LOGIN`, `channel info`, `needs_session false` — shown in full in §D1.19. Configuration comes from the same TTL provider descriptor ingestion reads: `mds.entity`, `mds.agreement`, `mds.language`, `client.device`, `client.dataDevice`, `client.version`, `websocket.info.url`; `mds.password`, `mds.key` and `mds.email` as secret-manager references (§B10.7).

<mark>The relay takes the mapped user as sent. With no password on the TTL MDS login, anyone able to call it can obtain a session for any user id.</mark>

<mark>To verify in UAT: the `sessID` stays valid when the socket that logged in is reused for other customers' frames or closed. If it does not, the Forward Handler keeps one info socket per active session.</mark>

### C1.3 Where the login code lives

The encryption already exists as `TtlTokenGenerator`, today in the ingestion core — listed as a deviation in §B10.12. It belongs in the TTL adapter, where ingestion and the Forward Handler share it; the Forward Handler exposes it to JSONata as `$loginToken(user)` (§C7.6).

## C2. Client API Gateway — **Drafted**

The single entry for client traffic. It turns an HTTP call into one `serviceCode` per unit of work, publishes it to the topic its configuration names, and assembles the replies. It never knows what a `serviceCode` means or how it is answered. Deployed as `market-api-gateway`.

### C2.1 From a request to a topic

![From a path to a topic](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/gwd-02-path-to-topic.png)

**Text alternative:** An HTTP request arrives with a method and a path. The gateway finds the rows of `gateway_route` matching that method and path. Many rows means a batch endpoint, and each body item carries the `serviceCode`, which must be one of that path's rows; one row means the endpoint is itself one `serviceCode`, which the client never sends. The request is then validated — row status, args schema, credential rule — and the topic follows the mode: `market.forward.request.v1` when `GATEWAY_MODE` is FORWARD, the row's `storage_topic` when it is STORAGE. The key comes from the row's strategy: `correlationId:itemIndex` for market data items, `requestId` for independent single calls, an `args` field for a login, the customer for a subscription.

| Endpoint | Dispatch | `serviceCode` | Detail |
|---|---|---|---|
| `POST /api/v1/common/market` | `BATCH` | Each item's, from the body | §B3, §B7 |
| `POST /api/v1/auth/login` | `SINGLE` | `auth.login` | §B7.2, §B11.6 |
| `GET /api/v1/packages` · `POST /api/v1/packages/subscribe` · `/cancel` | `SINGLE` | `package.*` | §B11.7 — STORAGE deployments |
| `GET /api/v1/market-data/stream` — WebSocket | `STREAM` | `stream.*` | §B4 — served by `market-stream` |

| # | Step | On failure |
|---|---|---|
| 1 | `(method, path)` → the `gateway_route` rows | None → `404`, nothing published |
| 2 | `BATCH`: each item's `serviceCode` must be a row of this path. `SINGLE`: the row's own | `400` for a malformed batch · item error `REQUEST_SERVICE_NOT_ALLOWED` |
| 3 | Row `ACTIVE`; `args` valid against `args_schema_ref`; `credential_rule` satisfied | `REQUEST_SERVICE_SUSPENDED` · `REQUEST_INVALID_ARGS` · `401 AUTH_TOKEN_MISSING` |
| 4 | Topic: `GATEWAY_FORWARD_TOPIC` in FORWARD, the row's `storage_topic` in STORAGE | STORAGE with no topic → `REQUEST_SERVICE_UNAVAILABLE` |
| 5 | Key by `key_strategy`; publish the §D2 `ServiceRequest` | `503 UPSTREAM_KAFKA_UNAVAILABLE` |
| 6 | Wait on the instance's own reply partition; assemble; apply `projection` | Unanswered at the deadline → `TIMEOUT_DOWNSTREAM` |

**Dispatch modes and key strategies are code — a closed set.** A row naming one that does not exist is refused when the configuration is published, not when the call arrives.

| Dispatch | Behaviour | | Key strategy | Key |
|---|---|---|---|---|
| `BATCH` | Many rows share the path; one message per item; the response assembled by `itemIndex` | | `CORRELATION_ITEM` | `correlationId:itemIndex` |
| `SINGLE` | The body is the `args`; one message; the one reply is the response | | `REQUEST_ID` | `requestId` — the client's `Idempotency-Key` if sent, else generated |
| `STREAM` | WebSocket upgrade; messages per session (§B4) | | `ARG` · `SUBJECT` | `args.{key_arg}` · `subjectRef` |

### C2.2 Dispatch is configuration — **Drafted**

![One table, one row per serviceCode](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/gwd-03-route-table.png)

**Text alternative:** Three HTTP endpoints on the left. The common market path carries many rows — `chart`, `symbolatest`, `company` — each naming its STORAGE topic and its key; the login path carries the single row `auth.login`; the subscribe path carries the single row `package.subscribe`. On the right, the columns of `gateway_route`, and the reminder that in FORWARD the `storage_topic` column is ignored and every row goes to `market.forward.request.v1`.

| | Lives in | Changes by |
|---|---|---|
| **The mode** and the fixed topics: `GATEWAY_MODE`, `GATEWAY_FORWARD_TOPIC`, `GATEWAY_REPLY_TOPIC`, `GATEWAY_REPLY_PARTITION` | The gateway's env file | A deployment setting |
| **Every `serviceCode`**: its path, dispatch, `storage_topic`, key strategy, credential rule, args schema, timeout and status | `mds.gateway_route` — DDL in §D1.19. Owned by **admin-service**, published as a compacted snapshot | A configuration change, maker-checker — no deploy |
| **How**: building a key, fanning out a batch, assembling a reply | Code — a closed set | A release |

The table serves the Client API Gateway only. Flows between services — the market data feed, configuration, admin commands — never pass through the gateway and have no row here (§B11.5).

**Checked when the configuration is loaded** — a row breaking one of these is rejected, and the gateway keeps the snapshot it already had

| # | Rule | Why |
|---|---|---|
| 1 | All rows of one path share the same `dispatch` | A path is either a batch or a single call |
| 2 | A `SINGLE` path has exactly one `ACTIVE` row | The gateway must know which `serviceCode` to label |
| 3 | A `serviceCode` is reachable only through its own path | `auth.login` cannot be smuggled inside the common market batch |
| 4 | `storage_topic` follows the §D2.4 naming | One convention across the platform |

![Where the table comes from](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/gwd-04-config-source.png)

**Text alternative:** Two bands. In the interim the gateway reads `mds.gateway_route` from PostgreSQL at startup and on every refresh interval, building an in-memory snapshot whose rules are checked before it is swapped in atomically. In the target admin-service owns the table and publishes it on the compacted topic `market.config.gateway-route.v1`, keyed by `serviceCode`; the gateway consumes that instead, with the same checks and the same swap.

<mark>Interim, until admin-service is built: the gateway reads the table directly from the database at startup and on a refresh interval.</mark> The request path never reads the database in either phase.

### C2.3 Responsibilities

| Does | Does not |
|---|---|
| Validate the request; check the required credential is present (§B7.3) | Validate a token in FORWARD, or issue one |
| Label each unit of work with its `serviceCode`; assign `correlationId`, `deadlineAt`, `replyTo`; publish | Know what a `serviceCode` means, or how it is answered |
| Wait for replies on its own partition; assemble one response; apply `projection` | Hold business state; retry a call that changes state |
| Rate limit; mask credentials in logs and traces | |

### C2.4 Replies — **Settled**

![A batch, answered out of order](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/gwd-06-batch-assembly.png)

**Text alternative:** A client batch of four items. The gateway publishes one message per valid item — m0, m1 and m3 — while the third item, whose args are invalid, fills its slot immediately with an error and is never published. The handlers answer in any order, each exactly once, on the gateway's own partition of the reply topic. Each reply finds its slot through `causationId`. The slots fill as replies arrive; when none is outstanding the gateway answers the client with the array in the order asked.

**What the gateway holds for one call, in memory**

| Field | Content |
|---|---|
| `correlationId` | One per HTTP call |
| `slots[n]` | The response array; a local error fills its slot at once and publishes nothing |
| `waiting` | `messageId → itemIndex`, for every published message not yet answered |
| `deadlineAt` | Absolute; the row's `timeout_ms` from arrival |

| Rule | |
|---|---|
| Topic | `market.gateway.reply.v1`, for every mode |
| Assignment | Static: each instance owns one partition and names it in `replyTo` (§D2.7) |
| One request, one reply | Every service answers every request exactly once — a failure is a reply with `isSuccess: false`. A provider's multi-frame answer is assembled before replying |
| Matching | By `causationId` — the request's `messageId`; `correlationId` and `itemIndex` are cross-checked |
| Complete | When `waiting` is empty, or at `deadlineAt` — each unanswered item gets `TIMEOUT_DOWNSTREAM` |
| Duplicate, late or unknown reply | Not in `waiting` → discarded and counted. The first reply wins |
| Timeout of a call that changes state | The outcome is unknown, not failed. The client repeats it with the same `Idempotency-Key`; the service returns the first outcome |
| Retention | One minute — a login reply carries a session id |

![One reply topic, one partition per gateway](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/gwd-07-reply-partitions.png)

**Text alternative:** Three gateway pods, each with its own `GATEWAY_REPLY_PARTITION`, read one partition each of `market.gateway.reply.v1`; spare partitions stand ready for more pods. Any handler reads `replyTo` from the request and writes the reply straight to that partition. Operations keep four things true: partitions at least the number of gateway pods, one pod per partition with an alert otherwise, compression for large replies, and one-minute retention with the topic readable only by the gateway.

## C3. Token reading — **Deferred**

FORWARD reads no token (§B1.4). This applies only to a deployment that must read the token's payload — STORAGE, or anything that later needs the Login Server's claims.

| Decided | |
|---|---|
| Format | The Login Server plans JWE and provides a secret key to decrypt the payload (D18) |
| Where | In the gateway, not a separate service — decryption needs no network call |
| Cost | Decrypt once per token; remember the result in each gateway's memory, keyed by a hash of the whole token, for no longer than the token's expiry. A shared Redis lookup would cost more than the decryption it saves |
| Key | A reference into the secret manager; never in configuration, a log or a status response |

## C4. Subscription & Entitlement Service — **Drafted**

Owns the package catalogue, subscription state, entitlement resolution and the charge exchange with the core. STORAGE deployments; deferred. Data model in §D1.3; lifecycle in §A6.3.

## C5. Route & Offering Control — **Drafted**

### C5.1 Control plane, not a request hop — **Settled**

The service owns the business logic behind availability: licence state, provider capability, effective dates. From these it decides which `serviceCode`s a deployment may serve, and that decision becomes the `status` of a `gateway_route` row (§D1.19), published as a versioned snapshot on a compacted topic. Where a `serviceCode` goes is not a business decision: it is the deployment's mode and the row's topic (§C2.2).

| Gain | Cost |
|---|---|
| One fewer Kafka message per item — a 50-item batch costs 100 messages, not 150 | A route change takes effect only when the snapshot reaches every instance — seconds, not instantly |
| A suspended service fails at the gateway, before any message is produced | The gateway must fail closed until it has installed a snapshot at startup |
| Matches the snapshot-and-apply pattern already running in the gateway | |

This holds because **availability is always configuration**. Provider failure is handled by the Forward Handler through retry and circuit breaking (§C7), not by re-routing a request in flight. If per-request routing on provider health is ever required, this service returns to the request path.

Blocked on: offering scope by exchange or market (D1); offering code format when depth does not apply (D2).

## C6. Query Services — **Open**

Blocked on: entitlement policy for reference and fundamental endpoints (D3).

## C7. Forward Handler and Provider Adapter — **Drafted**

Deployed as `forward-service`. The end-to-end flow and its contracts are in §B7.

### C7.1 Upstream identity — **Settled**

The forward call carries the **customer's own provider token**, as the client sent it — in the interim, the TTL MDS token in `MDS-Authentication` (§B1.3). The provider sees each customer individually and checks the token and the customer's rights itself. MDS validates nothing, checks no entitlement, and keeps no shared service-account pool.

| Consequence | |
|---|---|
| Entitlement | The provider's. MDS maps what comes back — fields present are mapped, fields absent are left empty |
| Per-customer record | Held by the provider, since each call arrives under the customer's own identity |
| Token transport | Forward topic only (§B1.5) |

### C7.2 Streaming multiplexing — **Drafted**

Where sharing is permitted, one upstream subscription per `(provider, symbol)` with reference counting — opened on the first subscriber, closed on the last.

**Open (D11):** if sharing is not permitted, a dedicated session per customer is required — a difference of orders of magnitude in cost. Because the provider checks each customer's own token, one shared upstream subscription could deliver data to a customer the provider never checked; sharing therefore needs the provider's explicit permission.

### C7.3 Failure semantics — **Settled**

A state that exists only in FORWARD mode: the request is well formed, but the upstream cannot serve — provider unreachable, session failure, or quota exhaustion.

Returns `UPSTREAM_ENTITLEMENT_UNAVAILABLE`. A refusal of the customer's rights by the provider is also returned as an `UPSTREAM_*` code. Raw provider errors never reach the client.

### C7.4 Pipeline — **Drafted**

One batch item passes through ordered stages. Code owns anything stateful or asynchronous; JSONata owns pure shape.

| # | Stage | Kind | Does |
|---|---|---|---|
| 1 | Deadline and enable guard | code | Past `deadlineAt` → drop. Exchange or service disabled → item error |
| 2 | Identity | code + cache | Platform `(XHKG, 00700)` → TTL `(HK, 700)` |
| 3 | Build | JSONata | TTL frame from the `serviceCode`'s `request_jsonata`, input `{instrument, args, provider: {device, version, entity, sessID}}`, with `sessID = providerToken` |
| 4 | Send | code | On the pool named by the row's `channel`; `reqId` per socket; wait up to `timeout_ms` |
| 5 | Reassemble | code | Frames for one `reqId` until `isLast`, per `multipart_mode` — `single`, `appendPath`, `collect` |
| 6 | Transform | JSONata | The assembled TTL document → the service's output shape |
| 7 | Reverse identity | code | TTL `(HK, 700)` → platform `(XHKG, 00700)` |
| 8 | Reply | code | Publish to the request's `replyTo` with the `itemIndex`; drop the token reference |

A bulkhead and circuit breaker per `bulkhead_group` keep slow reference calls from starving prices. The same stages serve a `serviceCode` that is not a market data query, such as `auth.login` (§C7.6); a row with `needs_session false` skips stage 2 and adds no `sessID` in stage 3.

### C7.5 Socket pools — **Drafted**

Two pools, as in ingestion: **info** for `SECURITY_*`, **snapshot** for prices and history.

| Rule | |
|---|---|
| Sockets | Transport to TTL MDS only. A pool never logs in; each frame carries its own customer's `sessID` |
| Correlation | Each socket has its own `reqId` counter and pending map; a reply returns on the socket it was sent on |
| Dispatch | Least in-flight first; round-robin on a tie |
| Scale | `min-sockets` 2 to `max-sockets` 20 per pool; up when every socket is above 80% of `max-inflight-per-socket` (50) for a sustained window; down after 120 s idle |
| Back-pressure | Every socket at its cap → wait up to the item's timeout, then `DOWNSTREAM_BUSY` |
| Liveness | Reuse the ingestion TTL socket channel — heartbeat, reconnect, counters |

### C7.6 Service codes that are not queries — **Drafted**

In a FORWARD deployment the Forward Handler answers every `serviceCode` the deployment serves, not only market data queries. Each is one `gw_operation_config` row on `market.forward.request.v1`, and the pipeline is the same (§B11.7).

| `serviceCode` | `msg_type` | `channel` | `needs_session` | Reply `data` |
|---|---|---|---|---|
| `auth.login` | `LOGIN` | info | false | `{ mdsToken, user, services }` — §B7.2 |
| Next, e.g. `package.list` | TTL's message | per TTL | true | Per the endpoint's contract |

**JSONata helpers — code, the only way a row reaches a secret or a computed value**

| Helper | Returns |
|---|---|
| `$loginToken(user)` | `hex(AES(entity:UTC-timestamp:user, MDS key))` — `TtlTokenGenerator` |
| `$providerSecret(name)` | A secret from the provider configuration, e.g. `mds.password`, resolved from the secret manager |
| `$mapWireServices(map)` | TTL's `service` map with wire exchange codes turned into platform codes |
| `$toTtlTime(iso)` | An ISO-8601 instant as TTL's time format |

A row's JSONata text never contains a secret value.

<mark>To verify in UAT: TTL accepts frames carrying different customers' `sessID` on one socket. Ingestion already uses one `sessID` across its info and snapshot sockets; if TTL instead binds a `sessID` to a socket, the pool is keyed by `sessID` — one socket per active customer, closed when idle.</mark>

## C8. Ingestion and the Provider SPI — **Drafted**

> Ingestion exists only in a **STORAGE** deployment. In FORWARD the provider connection belongs to `forward-service`, which answers requests (§C7) and fans out streaming by session key (§C10.3).

The service that owns every upstream market-data connection in STORAGE mode. Its structure is the concrete instance of the adapter rule in §B10; this section is the detail behind that rule.

### C8.1 Responsibilities — **Settled**

| Owns | Does not own |
|---|---|
| Provider sessions and their lifecycle | topic layout and routing — §D2 |
| Normalisation into the canonical model | storage: it publishes, it never writes to the database |
| Symbol universe resolution for subscription | entitlement and licence — §C4, §D5 |
| Crawl and reference job execution | which provider serves which exchange — that is published to it as configuration |
| Provider health and streaming status | the operator interface that changes configuration — §C11 |

### C8.2 Internal structure — **Settled**

| Layer | Package | Contains | Depends on |
|---|---|---|---|
| Core | `core/model`, `core/port`, `core/service`, `core/exception` | canonical model, ports, registry and factory, session supervision | nothing outside `core` |
| Provider adapters | `adapter/provider/<code>` | wire protocol, sessions, parsers, that provider's factory | `core` and that provider's SDK |
| Output adapter | `adapter/publisher` | the Kafka publisher | `core` |
| Infrastructure | `infra/config`, `infra/entity`, `infra/health` | snapshot intake, symbol universe loading, health reporting | `core` |
| API | `api/controller`, `api/dto`, `api/mapper` | the operational HTTP surface | `core` |

### C8.3 The provider SPI — **Settled**

| Interface | Method | Function |
|---|---|---|
| `AdapterManagerFactory` | `providerCode()` | Identifies the provider; the platform selects the factory by matching configuration |
| | `create(ProviderConfig)` | Builds the adapter from that provider's configuration |
| `AdapterManager` | `connect()` / `disconnect()` | Establishes and tears down the provider connection |
| | `status()` | Provider status for health and admin display |
| | `session()` | The session this manager owns |
| | `streaming()` / `crawl()` / `reference()` | Declares which capabilities this provider serves — §C8.4 |
| `MarketDataProvider` | `subscribe(exchange)` | Opens the upstream feed and starts streaming |
| | `unsubscribe(exchange)` | Stops streaming and releases the subscription |
| | `isStreaming(exchange)` | Reports current state for health checks and admin status |
| | `streamingExchanges()` | Lists all exchanges currently streaming |
| | `refreshMarket(exchange)` | Re-synchronizes after a configuration or symbol-universe change. Optional |
| `CrawlProvider` | `supportedKinds()` | Declares `HISTORICAL` / `FINANCIAL` support so the scheduler dispatches only servable work |
| | `crawl(request, job, sink)` | Executes one crawl job, emitting normalized records; returns the outcome for tracking and retry |
| `ReferenceProvider` | `fetchReferences(query, sink)` | Retrieves instrument reference data and emits normalized records |
| `ProviderSession` | `sessionId()` | Session identifier for correlation and diagnostics |
| | `isActive()` | Whether the session is usable; false triggers re-establishment |
| | `nextReqId()` | Allocates the next identifier for the provider's correlation scheme |
| | `invalidate()` | Marks the session dead so the supervisor rebuilds it and resubscribes |
| `EnvelopeSink` | `accept(MarketEnvelope<?>)` | The single output path; everything an adapter produces passes through it |

| Contract | Rule |
|---|---|
| Output | Adapters emit into `EnvelopeSink` using the standardized model. They never publish to Kafka and never choose a topic |
| Errors | Typed exceptions at the port boundary. An unmapped exchange or symbol raises an error rather than silently dropping data |
| Out of scope | Symbol and exchange-code translation resolves from configuration, so mapping rules change without a code release |

### C8.4 Capability declaration — **Settled**

`AdapterManager` returns each capability as an optional. Absence is the normal way to say "this provider does not do that": the scheduler simply does not dispatch, rather than calling and handling a failure. Rationale and the full table are in §B10.6.

### C8.5 Configuration intake and provider lifecycle — **Settled**

![Configuration intake and provider lifecycle](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-c8-provider-lifecycle.png)

**Text alternative:** An operator edits configuration in market-admin, which publishes one full desired state to a compacted snapshot topic. A validator checks the topic contract and the content; a rejection marks the gate in error rather than applying part of it. The apply coordinator installs the validated candidate, rebuilds the provider registry from it, and only then opens the gate. At runtime the registry holds one AdapterManager per enabled provider, each owning one provider session.

| Stage | Rule |
|---|---|
| Startup | The snapshot topic is replayed to its end before the service serves. The gate stays closed until it is caught up |
| Validation | Topic contract first, then content. A rejected snapshot leaves the previous state in place and marks the gate `ERROR` |
| Apply | A **full rebuild**, not a diff: install the candidate → shut down and reset providers → reload only enabled providers → enable only eligible exchanges → open the gate |
| No fallback | Never falls back to static configuration or to a previous work set. An unusable snapshot means a closed gate, not degraded guesswork |
| Secrets | References resolve into an ephemeral construction copy. The resolved value never re-enters stored configuration, a log, or a status response |
| Region | A snapshot is keyed by region and a deployment applies only its own. A deployment without a region identity refuses to apply anything |

**Why a full rebuild, and what it costs.** A diff has to be correct for every pair of states; a rebuild has one path, which is the difference between a design that survives its tenth configuration change and one that does not. The cost is real: applying a snapshot drops and re-establishes provider sessions, so a configuration change is a short streaming interruption. That is acceptable because configuration changes are operator actions, not traffic — but it does mean **configuration changes are scheduled, not casual**.

### C8.6 Symbol universe and the hot path — **Drafted**

| Concern | Design |
|---|---|
| Source | The symbol snapshot, held in memory. No per-frame database access |
| Outbound | `SymbolUniversePort.subscriptionKeys(provider, canonicalExchange, type)` returns provider-native subscription keys |
| Inbound | A resolved-symbol index keyed by **wire exchange and provider symbol**, because an inbound frame carries the provider's vocabulary, not ours |
| Readiness | Streaming is gated on the universe being loaded; a partially-loaded universe would subscribe to a partial market silently |
| Scale | The file-backed implementation is a startup watchlist of a few dozen symbols for test convenience only. A full exchange universe — HKEX alone is roughly 31,000 symbols — needs the cache-backed implementation of the same port, which is an infrastructure change with no effect on `core` |

### C8.7 Envelope construction — **Settled**

| Field | Source | Rule |
|---|---|---|
| `eventId` | provider event id, or generated | the deduplication key for downstream consumers |
| `provider` | provider code | lineage and diagnosis only — never a behavioural input |
| `exchange` | `CanonicalExchangeResolver` | MIC form. Unmapped raises a typed error and the record is not published |
| `symbol` / `providerSymbol` | symbol mapping | both are carried: canonical for consumers, provider-native so a bad record can be traced upstream |
| `eventType` | the adapter | `REFERENCE` · `TRADE` · `QUOTE` · `FINANCIAL` · `BAR` |
| `eventTime` / `ingestTime` | provider / platform | both UTC, both kept, so ingestion lag is measurable without a second source |
| `schemaVersion` | constant | additive model changes increment it; consumers tolerate unknown fields |
| `body` | the adapter | one canonical record type; no provider vocabulary in field names or values |

### C8.8 Output and topic selection — **Settled**

The publisher adapter chooses topic and key from the event type and the canonical `exchange:symbol`. No adapter knows a topic name. Topic conventions are in §D2.

### C8.9 Crawl and reference jobs — **Drafted**

| Kind | Trigger | Contract |
|---|---|---|
| Historical bars | operator command or schedule | one job, many records, one outcome |
| Financial statements | operator command or schedule | same |
| Instrument reference | reference refresh | `fetchReferences(query, sink)` — records stream into the same sink |

| Rule | Reason |
|---|---|
| Failure is a returned outcome, not a thrown exception | retry is the scheduler's decision, made with the whole job's result in view |
| One job may emit many records through the sink | back-pressure and partial progress are visible rather than buffered in memory |
| Interactive login — a provider that requires a captcha or a human step — is an **operator action on the admin path**, never on an automatic schedule | an automatic job that can block on a human is an outage waiting for the weekend |

### C8.10 Health and failure — **Drafted**

| Failure | Detection | Response |
|---|---|---|
| Session dead | `isActive()` false, or `invalidate()` called by the adapter | the supervisor rebuilds the session and resubscribes the exchanges that were streaming |
| Provider unreachable | `status()` | health reports degraded; affected markets report not-streaming rather than silently empty |
| Unmapped exchange or symbol | typed exception at the port boundary | the record is rejected and alerted; it is never published with a provider code in it |
| Snapshot rejected | validator | the gate goes `ERROR` and the previous state is retained |

The common thread: **a failure is always visible as a state, never as an absence of data.** An empty market and a broken feed must not look the same to an operator.

### C8.11 Deployment and scaling — **Drafted**

| Property | Design | Consequence |
|---|---|---|
| Region | One deployment per region, applying only its region's snapshot | one region's configuration cannot disturb another |
| Provider session | Stateful — one manager and one session per provider per deployment | two active instances of the same provider would open two upstream subscriptions, doubling metered usage and duplicating every event |
| Redundancy | Therefore active/standby per provider, not active/active, unless a provider's contract permits duplicate subscriptions | **Open — D15** |
| Crawl work | Stateless per job | can scale out independently of streaming |

### C8.12 Conformance test suite — **Drafted**, not yet built

| # | Layer | Checks |
|---|---|---|
| 1 | SPI conformance | Subscribe/unsubscribe idempotency, state consistency, reconnection |
| 2 | Golden tests | Recorded upstream payloads with expected normalized output committed as files |
| 3 | Cross-provider invariants | UTC and monotonic timestamps, canonical symbol and exchange resolution, required fields, typed error on unmapped exchange |
| 4 | Client contract | Provider field coverage asserted against the public DTO |

**Regression protection for the Nth market:** every existing provider's golden files are committed artefacts re-run in CI. A new market cannot alter an existing market's output without a failing build.

Delivered with the ICE adapter, so Korea is the first adapter to pass it.

## C9. Processing and Delayed Delivery — **Open**

## C10. Streaming Distribution — **Drafted**

Deployed as `market-stream`. It holds the client connections, decides what each session may see, and writes the frames. The end-to-end flow and the client protocol are in §B4.

### C10.1 Session load model — **Settled**

Applies to STORAGE deployments, where MDS owns entitlement. In a FORWARD deployment there is nothing to revalidate on a timer: the provider answers `401` at the next command (§B4.4).

Revalidation is per **session**, not per subscription, so it does not scale with symbol count.

```
validation requests/sec = concurrent sessions ÷ revalidation interval
```

| Concurrent sessions | Req/s at 30 s |
|---|---|
| 10,000 | ~333 |
| 50,000 | ~1,667 |
| 100,000 | ~3,333 |
| 200,000 | ~6,667 |

**Open (D13):** target concurrency not supplied. The interval is a tuning lever — 30 s to 60 s halves the load, at the cost of worst-case revocation delay.

### C10.2 Renewal, by mode — **Settled**

Whether a connection survives a renewal depends on one question: does the identity the stream is keyed by change?

| | FORWARD | STORAGE |
|---|---|---|
| What the client presents | The **provider's** session — it is the credential | A platform token over a session MDS owns |
| Does renewal change it | Yes. A new provider session is a new id, a new hash and a new partition | No. The subject stays the same, so the key and the partition stay |
| The connection | **Rebuilt** — closed with `AUTH_TOKEN_EXPIRED`, then the client logs in and reconnects (§B4.4) | **Kept** — one in-band frame carries the new token; subscriptions are never re-established |
| Periodic revalidation | None; the provider refuses at the next command | Server-side against local state, never touching the client |

**Why FORWARD does not try to keep the connection.** Carrying a live connection across a change of identity would mean a second key for the socket, a state kept for a session that no longer exists, and a recovery path used by nothing else. A client must already survive a lost socket — a deploy, a network fault, a lost instance — so an expiry reuses that one path.

| Gain | Cost |
|---|---|
| One recovery path, one key, no grace window and no connection state to migrate | A re-login costs a reconnect and a fresh subscribe, and the client re-sends its symbol list |
| An expiry is indistinguishable from any other lost socket, so the client has one branch to test | A user switching devices makes both devices reconnect (§B4.4) |

### C10.3 Fan-out is Kafka partitioning — **Settled**

Scale comes from partitions, not from a socket cluster. The session key of §B4.2 is the message key on both streaming topics, so every message of one session lands on one partition, and one instance owns that partition.

| Rule | |
|---|---|
| Assignment | Static, as for replies (§D2.7): instance `i` of `M` owns every partition where `p % M == i` |
| Partition count | `STREAM_PARTITIONS` — fixed at 64, the ceiling on instance count |
| Ingress | Routes on the `p` the login reply gave the client. A socket on the wrong instance is closed with `STREAM_WRONG_NODE` |
| Session registry | In memory: `sessionKey → { connectionId, sessionID, subscriptions, sequence }`. Never in Redis, never on Kafka |
| Ordering | One session's subscribe, unsubscribe, close and delivery messages share a key, so they keep their order |
| Teardown | Every ended socket sends one idempotent `stream.close` on `market.stream.command.v1`, so `forward-service` unsubscribes upstream and the provider subscription never outlives the socket (§B4.5) |
| Heartbeat | Each instance beats every 5 s on the same topic. `forward-service` groups its upstream state by message `source`; a source silent for three beats has its sessions dropped — the safety net for an instance that dies before it can send a teardown |

| Gain | Cost |
|---|---|
| A message reaches the instance holding the socket without a lookup or a second hop | Changing the instance count moves partitions, and those sockets reconnect |
| No shared session store to keep consistent, and nothing to replicate between instances | The ingress must route on the same `p` the login reply gave the client |
| An instance failure is contained: only its own sockets reconnect | `P` must be set generously at creation — partitions can be added but not removed |

**Who produces the stream, by mode**

| Mode | Producer | What it publishes |
|---|---|---|
| **FORWARD** | `forward-service` | It holds the customer's provider session, subscribes upstream and **fans out keyed by `sessionKey`** on `market.stream.delivery.v1`. Ingestion does not exist in this deployment |
| STORAGE | Ingestion → Processing → the instrument topics | Each `market-stream` instance reads every partition and filters to its own sessions; there is no per-session republication |

### C10.4 Transports are configuration — **Drafted**

One core, adapters at the edge. Which adapters run, and which broker they use, is env (§B4.8).

```env
STREAM_TRANSPORTS=websocket
STREAM_WS_PATH=/api/v1/market-data/stream
STREAM_PARTITIONS=64
STREAM_INSTANCE_ORDINAL=0
STREAM_PING_INTERVAL=15s
STREAM_HEARTBEAT=5s
# STREAM_TRANSPORTS=websocket,mqtt
# STREAM_MQTT_BROKER=tcp://emqx:1883
```

| In the core — every transport | In the adapter — per transport |
|---|---|
| Session registry and subscription index | Frame encoding and field names |
| Entitlement at subscribe, and revalidation | How a per-symbol error is reported |
| Realtime or delayed selection | Close codes, heartbeat, handshake |
| Conflation, rate capping, slow-consumer cutoff | Topic naming and broker ACL — MQTT only |
| Reading Kafka and filtering by symbol | |

The FORWARD deployment runs WebSocket alone: a customer's stream is their own provider session, so there is no shared dataset to publish per instrument. MQTT earns its place in STORAGE.

## C11. Admin Services — **Open**

Blocked on: admin identity source (D7); maker-checker state model.

---

# Part D — Cross-cutting Reference

## D1. Data Model — **Drafted**

Specified **structurally**: **Essential** attributes carry meaning the design depends on; **Optional** attributes are left to implementation. Column-level detail, types and constraints live in `market-data-schema-design.md` and the migration under `market-data-db-migration`.

### D1.1 Cluster map

![Data model cluster map](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-erd-00-cluster-map.png)

| # | Cluster | Entities | State |
|---|---|---|---|
| 1 | Reference backbone | 8 | Implemented |
| 2 | Market calendar | 2 | Implemented · 2 extensions required |
| 3 | Product taxonomy and order types | 10 | Implemented |
| 4 | Instrument master | 4 | Implemented |
| 5 | Instrument attributes | 4 | Implemented |
| 6 | Classification | 4 | Implemented |
| 7 | Fundamentals and corporate actions | 4 | Implemented |
| 8 | Time-series | 6 | Implemented |
| 9 | Provider and routing configuration | 3 | Implemented · 2 new |
| 10 | Offering and service catalogues | 3 | **New** |
| 11 | Identity | 2 | **New** |
| 12 | Commerce | 4 | **New** |
| 13 | Audit and licence reporting | 2 | **New** |

46 entities exist in the migration today. 13 are new for this scope: identity, commerce, the offering and service catalogues, and the audit tables.

### D1.2 Conventions — **Settled**

| Convention | Rule |
|---|---|
| **Timestamps** | Every genuine instant is stored in UTC. The two schedule tables are the deliberate exception: they hold local wall-clock time and local dates, interpreted through `exchange.timezone` |
| **Identifiers** | Small controlled vocabularies use short natural keys (`currency_id`, `exchange_id`, `product_type_id`). High-cardinality entities use surrogate keys. Association tables use composite keys of their parents |
| **Internationalization** | Display names live in `*_translation` tables keyed by (entity, language), so core rows stay language-neutral |
| **Partitioning** | High-volume time-series partition by resolution then by time. A partitioned key must include the partition column |
| **Soft state** | Reference tables carry an `active` flag and audit timestamps rather than hard deletes |

### D1.3 Cluster 1 — Reference backbone

![Reference backbone and trading calendar](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-erd-01-reference-core.png)

| Entity | Essential | Optional |
|---|---|---|
| `language` | language id | display name, active |
| `currency` | currency id, minor unit | display name, active |
| `region` | region id, default currency, default language | display name, active |
| `market` (+ translation) | market id | display name, active |
| `exchange` (+ translation) | exchange id, **IANA timezone**, base currency, market | country code, trading days, venue type |

**Region and market have no direct key.** A region reaches a market through `region_exchange_access` and then `exchange`, because a market-data licence is granted per exchange, not per market. A region serving Hong Kong, Greater China, the US, Korea and crypto holds one access row per exchange — eleven rows producing five markets. The market list is **derived, never stored**:

```sql
SELECT DISTINCT e.market_id
FROM region_exchange_access r JOIN exchange e USING (exchange_id)
WHERE r.region_id = ? AND r.active;
```

Storing a `region_market` table would create a second source of truth that can drift from the access rows the licence actually covers.

**Naming correction required.** The seeded market `SZ` is named "Greater China" and contains both `XSHG` (Shanghai) and `XSHE` (Shenzhen). `SZ` is Shenzhen's own code, so the identifier contradicts its contents — it should be `CN`.

**Stock Connect — Settled for phase one.** Northbound access from Hong Kong into Shanghai and Shenzhen is treated as ordinary access to `XSHG` and `XSHE`. It is genuinely an access channel with its own eligible-security list and quota rather than a market, but the distinction only matters for order placement, which is outside MDS. If eligibility filtering is later required, it becomes an instrument attribute (§D1.7), not a new market — a separate market would put one instrument under two markets and break uniqueness on `(exchange, symbol)`.

**Extension required.** `exchange` needs a `venue_type` (`LISTING` / `MTF` / `ATS` / `OTC`). The table currently conflates listing venues with execution venues — a distinction that matters for markets where a security is listed on one exchange and traded on many.

### D1.4 Cluster 2 — Market calendar — **Settled**

| Entity | Essential | Optional |
|---|---|---|
| `exchange_trading_session` | exchange, session code, session type (`AUCTION` / `CONTINUOUS_TRADING` / `BREAK`), start and end local time, **validity dates** | sequence number, active |
| `exchange_holiday` | exchange, local date, day status, **close-time override** | description |

| Requirement | Covered by |
|---|---|
| Open and close times | Earliest session start, latest session end |
| Lunch break | A `BREAK` session |
| Auction windows | `AUCTION` session type |
| Daylight saving | IANA timezone plus local times — no stored offset to go stale |
| Half-day close | **Close-time override — extension required** |
| Historical session structure | **Validity dates — extension required** |

### D1.5 Cluster 3 — Product taxonomy and order types

| Entity | Essential | Optional |
|---|---|---|
| `exchange_product_category` (+ translation) | category id | active |
| `exchange_product_type` (+ translation) | product type id, category, requires-expiry flag, requires-strike flag | default lot size, active |
| `region_category_config` | region, category | active |
| `region_product_type_config` | region, product type, policy flags (short sell, odd lot, pre-market, after-hours) | active |
| `order_type` (+ translation) | order type id, requires-limit-price flag, requires-trigger-price flag | active |
| `time_in_force` | tif id | active |
| `product_type_order_type` | product type, order type | active |
| `product_type_time_in_force` | product type, tif | active |

### D1.6 Cluster 4 — Instrument master

![Instrument master](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-erd-02-instrument-master.png)

The central entity. Everything time-series and everything entitlement-scoped resolves through it.

| Entity | Essential | Optional |
|---|---|---|
| `instrument` | instrument id, exchange, product type, **symbol — unique per exchange**, status, quote currency | segment MIC, ISIN, lot size, tick size, expiry, strike, underlying instrument, listed and delisted dates |
| `instrument_translation` | instrument, language, display name | — |
| `instrument_provider_mapping` | instrument, provider, provider symbol — **unique per (provider, symbol)** | active, audit timestamps |
| `instrument_index_constituent` | index instrument, constituent instrument | weight, effective date |

**Index required.** `instrument_provider_mapping` is queried by `provider_symbol` alone during tick resolution, but the only index leads with `provider_code` — so that lookup cannot use it. See §D4.4.

### D1.7 Cluster 5 — Instrument attributes

![Instrument attributes](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-erd-03-instrument-attributes.png)

Product-type-specific fields without schema change. Adding an attribute is an insert.

| Entity | Essential | Optional |
|---|---|---|
| `instrument_attribute` | attribute code, value type (`STRING` / `DATE` / `DECIMAL`), **is-daily flag** | display name, active |
| `product_type_attribute` | product type, attribute code, required flag | active |
| `instrument_attribute_value` | instrument, attribute code, one typed value | updated at |
| `instrument_daily_attribute_value` | instrument, attribute code, **trade date**, decimal value | source · **partition by trade date** |

The `is_daily` flag separates static instrument facts from values republished each trading day, such as price bands and settlement prices.

#### Why an attribute table rather than columns

A global platform cannot express every market's instrument fields as columns. HKEX warrants need issuer and conversion ratio; KRX equities need a price band; options need exercise style; crypto needs a base asset and settlement currency. Adding a market would otherwise mean a migration on the busiest table in the system.

The trade-off is deliberate: **adding an attribute becomes a data insert, and screener queries lose their natural index direction.** The rest of this section is how the second half is paid for.

#### Sizing — the premise matters

An instrument does **not** hold a row per catalogue entry. `product_type_attribute` declares which attributes apply to each product type, so a stock carries roughly 12–15 rows while the catalogue itself spans every product type. Today the catalogue holds 53 codes, 19 of them daily.

| Table | Rows | Heap | PK index | Total |
|---|---|---|---|---|
| `instrument_attribute_value` — realistic, 100k instruments × ~15 applicable | 1.5 M | ~110 MB | ~57 MB | **~165 MB** |
| `instrument_attribute_value` — worst case, × 100 attributes each | 10 M | ~720 MB | ~380 MB | **~1.1 GB** |
| `instrument_daily_attribute_value` — per year | ~140 M | ~11 GB | ~6 GB | **~17 GB / year** |

Estimates from ~72 and ~78 byte rows; to be confirmed against a loaded environment.

Daily volume assumes ~16,000 equities carrying 16 financial attributes plus ~100,000 instruments carrying 3 price-band attributes, over 250 trading days.

**The static table is not the problem.** Even the worst case fits in memory on any server this platform would run on. **The daily table is** — it grows without bound and is the one that needs partitioning and retention.

#### Query patterns and what each costs

The primary key is `(instrument_id, attribute_code)`, so the leading column decides which queries are cheap.

| # | Pattern | Uses the key? | Estimated warm cost | Required index |
|---|---|---|---|---|
| 1 | All attributes of one instrument — `WHERE instrument_id = ?` | Yes, leading column | **< 1 ms** — one index descent, rows adjacent in the leaf | PK |
| 2 | One attribute for N instruments — `WHERE instrument_id = ANY(...) AND attribute_code = ?` | Yes | **~1–2 ms** for 200 instruments | PK |
| 3 | **Screener — `WHERE attribute_code = ? AND value_string = ?`** | **No** | **150 ms at 1.5M rows, 1–2 s at 10M — full scan** | **Partial indexes below** |
| 4 | Daily value over a date range — `WHERE instrument_id = ? AND attribute_code = ? AND trade_date BETWEEN ...` | Yes, full prefix plus range | **< 1 ms** | PK |
| 5 | Cross-sectional daily — `WHERE attribute_code = ? AND trade_date = ?` | No | Full scan without partitioning | Partition pruning plus index below |

Pattern 3 is the one that justifies the concern. EAV inverts the direction a screener wants to travel: the key leads with the instrument, but the screener starts from the value.

#### Indexes that make pattern 3 and 5 viable

```sql
-- One partial index per value type. Exactly one value column is populated per row,
-- so the three indexes together hold about one table's worth of entries, not three.
CREATE INDEX ix_iav_code_string  ON instrument_attribute_value (attribute_code, value_string)
    WHERE value_string  IS NOT NULL;
CREATE INDEX ix_iav_code_decimal ON instrument_attribute_value (attribute_code, value_decimal)
    WHERE value_decimal IS NOT NULL;
CREATE INDEX ix_iav_code_date    ON instrument_attribute_value (attribute_code, value_date)
    WHERE value_date    IS NOT NULL;

-- Cross-sectional daily reads, within a pruned partition.
CREATE INDEX ix_idav_code_date   ON instrument_daily_attribute_value (attribute_code, trade_date);
```

With these, pattern 3 becomes an index range scan on a narrow slice and lands in **single-digit milliseconds**. Pattern 5 prunes to one day's partition first, then uses the index within it.

#### Keeping the daily table bounded

| Control | Effect |
|---|---|
| **Partition by `trade_date`, monthly** | A date-ranged read touches one or two partitions instead of the whole table; retention becomes a partition drop rather than a mass delete |
| **Retention per attribute class** | Price-band values age out quickly; financial ratios are kept long. Configured through the retention policy, not hard-coded |
| **Cold tier** | Partitions past the warm window move to ClickHouse alongside the rest of the historical data |

Without partitioning this table reaches roughly 17 GB in the first year and keeps growing. Partitioning is not an optimisation here; it is what makes retention possible at all.

#### The cache absorbs the dominant read

Pattern 1 is by far the most frequent — every instrument detail response needs it — and attribute values change rarely. It is served from Redis, not from PostgreSQL.

| | |
|---|---|
| Key | One hash per instrument holding its full attribute set |
| Invalidation | On write, by the same service that writes the row |
| Effect | In steady state pattern 1 does not reach the database at all; the table serves cache misses, screeners and administration |

This is the same reasoning as the snapshot path: the store of record is PostgreSQL, the read path is Redis.

#### When to stop using the attribute table

The design has a boundary, and it should be crossed deliberately rather than discovered under load.

| Signal | Action |
|---|---|
| One attribute is filtered or sorted on by nearly every screener query | Promote it to a real column on `instrument` and backfill. The attribute table keeps the long tail |
| An attribute needs a foreign key, a check constraint, or referential integrity | Promote it — EAV cannot express those |
| A query needs several attributes as filters at once | Each becomes a self-join. Beyond two or three, promote them or build a materialised projection |

Promotion is expected over time and is not a design failure. The attribute table exists so that a new market can be onboarded without a migration, not so that every field stays there forever.

### D1.8 Cluster 6 — Classification

![Sector and theme classification](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-erd-04-classification.png)

Self-curated hierarchy, independent of any provider taxonomy.

| Entity | Essential | Optional |
|---|---|---|
| `classification_scheme` | scheme id, category | source, active |
| `classification_node` (+ translation) | node id, scheme, parent node, code, level | active |
| `instrument_classification` | instrument, node, is-primary flag | assigned by, assigned at |

**Index required.** The screener queries node → instruments, but the key leads with instrument. See §D4.4.

### D1.9 Cluster 7 — Fundamentals and corporate actions

![Fundamentals and corporate actions](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-erd-05-fundamentals.png)

| Entity | Essential | Optional |
|---|---|---|
| `fundamental_metric` (+ translation) | metric code, metric group | unit, active |
| `instrument_fundamental_value` | instrument, metric, fiscal period type, fiscal year, value | period end date, currency, source |
| `corporate_action` | instrument, action type, ex date | record date, payment date, ratio, amount, currency, announced at |
| `instrument_trading_halt` | instrument, halt start | halt end, reason, source |

**Two constraints to note.**

`instrument_fundamental_value` has no restatement support — a corrected figure overwrites the original. Acceptable for serving current fundamentals; insufficient if research or backtesting later needs point-in-time correctness, which would require an `as_of` dimension in the key.

Adjusted prices are derived, not stored: MDS keeps unadjusted prices plus corporate-action factors and applies them at query time (§D1.16).

### D1.10 Cluster 8 — Time-series

![Time-series tables](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-erd-06-timeseries.png)

| Entity | Essential | Optional | Partitioning |
|---|---|---|---|
| `instrument_snapshot` | instrument, last price, event time | OHLC, volume, turnover, order book, delayed flag | none — one row per instrument |
| `ohlc_bar` | instrument, **resolution**, **bar time (UTC)**, OHLC | volume, turnover | LIST(resolution) → RANGE(bar time) |
| `index_breadth_bar` | index instrument, resolution, bar time, advance / decline / unchanged counts | ceiling and floor counts | same as `ohlc_bar` |
| `market_trade_raw` | event id, instrument, **event time**, price, volume | trade id, aggressor side, provider | RANGE(event time) |
| `market_quote_raw` | event id, instrument, **event time**, bid and ask | sizes, depth, provider | RANGE(event time) |
| `market_ranking_snapshot` | region, category, ranking type, snapshot time, elements | — | none — **partitioning required** |

**Rules.**

| Rule | Reason |
|---|---|
| Raw tick tables carry **no provider payload column** | The original payload lives only transiently in the Kafka envelope |
| Dedup is `ON CONFLICT (event id, event time) DO NOTHING` | At-least-once delivery; the partition key must be in the unique key |
| `bar_time` is always UTC | For daily bars it is midnight UTC of the exchange-local trade date, computed upstream |
| Realtime and delayed snapshots live in **separate Redis namespaces**, not separate tables | §A9 |

**Three sizing decisions required — see §D4.4.**

### D1.11 Cluster 9 — Provider and routing configuration

![Provider configuration and routing](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-erd-07-provider-routing.png)

| Entity | Essential | Optional | State |
|---|---|---|---|
| `provider_config` | provider code, enabled flag, configuration blob | display name | Implemented |
| `provider_exchange` | provider, canonical exchange, wire market code, wire stream code, crawl and streaming flags | active, audit timestamps | Implemented |
| `region_exchange_access` | region, exchange, **provider**, priority, streaming-enabled flag | active, audit timestamps | Implemented · **key change required** |
| `provider_capability` | provider, exchange, dataset, depth, access method, timing, licence reference, effective dates | notes | **New** |
| `provider_user_binding` | platform customer, provider, upstream identity, **credential reference** | status, linked at | **Not needed — a forwarded call carries the customer's own token (§C7.1)** |

**Key change.** `region_exchange_access` is keyed `(region, exchange)`, which permits only one provider per pair and makes its own `priority` column unusable. The key becomes `(region, exchange, provider)` with a partial unique constraint allowing one active primary per pair.

**A credential is never stored in these tables** — only a reference into the secret manager.

### D1.12 Cluster 10 — Offering and service catalogues — **New**

![Offering and service catalogues](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-erd-08-catalogues.png)

| Entity | Essential | Optional |
|---|---|---|
| `offering` | offering code, market, dataset, depth, access method, delivery timing, lifecycle status | display name, description |
| `offering_version` | offering, version, effective interval, **immutable published definition** | approval evidence |
| `gateway_route` | service code, status, path and dispatch, storage topic, key strategy, credential rule, args and output schema references, required offerings | display name, description, deprecation date, rate-limit class — DDL in §D1.19 |

Both catalogues follow `DRAFT → ACTIVE ⇄ SUSPENDED → RETIRED` with maker-checker on publish and suspend, and both publish as versioned snapshots on compacted Kafka topics (§D2.5).

**Where the line sits.** What is operational — which offerings and services are live, where they route, what they require — is data. What is structural — the shape of arguments and output — is code, because a service cannot be switched on if its handler does not exist.

### D1.13 Cluster 11 — Identity — **New**

![Identity, commerce and audit](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-erd-09-commerce.png)

| Entity | Essential | Optional |
|---|---|---|
| `customer` | customer id, **user class** (`RETAIL` / `PROFESSIONAL`), status | display name, locale |
| `external_identity` | customer, IdP code, IdP subject — **unique per IdP** | linked at, last seen |

No credential is stored, and no token is persisted in any form (§A5.3). STORAGE deployments only. `user_class` is essential because exchange declarations count and price the two classes separately.

### D1.14 Cluster 12 — Commerce — **New**

| Entity | Essential | Optional |
|---|---|---|
| `package` | package code, billing cycle, lifecycle status | display name, description, grace-period days, sort order |
| `package_offering` | package, offering | effective dates |
| `customer_subscription` | customer, package, effective from, state | effective to, source reference, auto-renew flag |
| `subscription_event` | subscription, event type, occurred at | actor, reason, reference to a core payment record |

Price, currency and tax are deliberately absent (§A6.2). `subscription_event` is **append-only** — it is the evidence a licence declaration is defended with.

### D1.15 Cluster 13 — Audit and licence reporting — **New**

| Entity | Essential | Optional |
|---|---|---|
| `daily_access_summary` | customer, exchange, depth, delivery timing, **trade date** | first and last access time, request count |
| `token_decision_audit` | decision time, **hashed** subject, offering, decision, reason | correlation id, gateway instance |

`daily_access_summary` is written **once per customer per exchange per day**, not per request. At 100,000 customers across five exchanges the ceiling is roughly 15M rows per month; realistic volume is closer to 2M. Both retain comfortably for a seven-year audit period with monthly partitioning.

Written in STORAGE deployments only (§A8).

### D1.16 Derived data is not stored — **Settled**

| Value | Treatment |
|---|---|
| Adjusted prices | Unadjusted prices plus corporate-action factors, applied at query time. A new corporate action never rewrites history |
| Technical indicators | Not stored. Consumers compute from raw bars |
| Computed fundamental ratios | Not stored where the inputs are present |

The rule: an entity earns a table only if it is a raw fact that cannot be derived from another table **and** has a real source feeding it.

### D1.17 Retention

| Data | Tier | Store |
|---|---|---|
| Raw tick | 1 month | PostgreSQL, then dropped |
| Historical — hot | recent | Redis |
| Historical — warm | recent months | PostgreSQL |
| Historical — cold | long range | ClickHouse |

Tier boundaries are configurable per dataset. MinIO and a dedicated TSDB are **not** in the baseline.

### D1.18 Open items

| # | Item | Decision required |
|---|---|---|
| D9 | **Cross-listed securities.** A+H pairs, ADRs and dual counters share issuer-level facts — fundamentals, corporate actions, classification — while having genuinely different prices. A `security` tier above `instrument` would let those facts be stored once | Introduce now, or defer until a cross-listed instrument enters the universe. Cheap now, expensive after fundamentals accumulate |
| — | **Restatement of fundamentals** | Add an `as_of` dimension, or accept overwrite |
| — | **Crypto price precision** | Current numeric scale cannot represent sub-satoshi values; required before crypto markets go live |
| — | **`provider_user_binding`** | Not needed: a forwarded call carries the customer's own provider token (§C7.1) |

### D1.19 Cluster 14 — Gateway and forward configuration — **New**

![Gateway and Forward Handler configuration](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/mds-erd-10-gateway-config.png)

**Text alternative:** Two groups of tables. The Client API Gateway reads one: `gateway_route`, one row per `serviceCode`, giving the path it is reached through, its dispatch, its STORAGE topic, its key strategy, its credential rule, its args schema, its timeout and its status; beside it the rules checked when the configuration is loaded. The Forward Handler reads `gw_operation_config`, `gw_exchange_map`, `gw_symbol_rule` and `gw_enablement`, which say what each FORWARD `serviceCode` is and how the provider answers it. The groups are never joined: `gw_operation_config.service_code` matches `gateway_route.service_code` by name.

| Tables | Read by | Published on | They say |
|---|---|---|---|
| `gateway_route` | Client API Gateway | `market.config.gateway-route.v1` | Where each `serviceCode` goes, and with what key (§C2.2) |
| `gw_operation_config`, `gw_exchange_map`, `gw_symbol_rule`, `gw_enablement` | Forward Handler | `market.config.forward-mapping.v1` | What each FORWARD `serviceCode` is and how the provider answers it (§C7) |

The mode and the fixed topics are not tables: they are the gateway's env — `GATEWAY_MODE`, `GATEWAY_FORWARD_TOPIC`, `GATEWAY_REPLY_TOPIC`, `GATEWAY_REPLY_PARTITION`.

All tables are in schema `mds`, owned by admin-service with maker-checker, and published as compacted snapshots (§B11.5). A service caches its tables in memory and never reads them on the request path.

<mark>Interim, until admin-service is built: each service reads its tables straight from the database at startup and on a refresh interval. Switching to the snapshot topics changes where the service loads from, not the tables.</mark>

**Client API Gateway**

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
    args_schema_ref  varchar(120) NOT NULL,      -- code-resident JSON Schema (§B3.8)
    output_schema_ref varchar(120),              -- the service's output shape, for the common API
    required_offerings text[],                   -- STORAGE only
    timeout_ms       integer      NOT NULL DEFAULT 5000,
    status           varchar(20)  NOT NULL DEFAULT 'DRAFT',   -- DRAFT | ACTIVE | SUSPENDED | RETIRED
    rate_limit_class varchar(30),
    description      varchar(200),
    version          bigint       NOT NULL DEFAULT 1,
    updated_at       timestamptz  NOT NULL DEFAULT now(),
    updated_by       varchar(60),
    CHECK (dispatch IN ('BATCH', 'SINGLE', 'STREAM')),
    CHECK (key_strategy <> 'ARG' OR key_arg IS NOT NULL)
);
CREATE INDEX gateway_route_path ON mds.gateway_route (http_method, path);
```

Rules checked when a snapshot is loaded are in §C2.2: one dispatch per path, one `ACTIVE` row on a `SINGLE` path, a `serviceCode` reachable only through its own path, and `storage_topic` following §D2.4.

**Seed for the HK / US FORWARD deployment** — the rows a developer needs to run the flow end to end. `storage_topic` is filled so the same rows serve a STORAGE deployment.

```sql
INSERT INTO mds.gateway_route
  (service_code, http_method, path, dispatch, storage_topic, key_strategy, key_arg,
   credential_rule, args_schema_ref, output_schema_ref, timeout_ms, status) VALUES
  ('symbolatest', 'POST', '/api/v1/common/market', 'BATCH', 'market.query.snapshot.request.v1',
   'CORRELATION_ITEM', NULL, 'MDS_AUTH_REQUIRED', 'SymbolAtestArgs', 'SymbolAtestOutput', 5000, 'ACTIVE'),
  ('chart',       'POST', '/api/v1/common/market', 'BATCH', 'market.query.timeseries.request.v1',
   'CORRELATION_ITEM', NULL, 'MDS_AUTH_REQUIRED', 'ChartArgs', 'ChartOutput', 5000, 'ACTIVE'),
  ('company',     'POST', '/api/v1/common/market', 'BATCH', 'market.query.reference.request.v1',
   'CORRELATION_ITEM', NULL, 'MDS_AUTH_REQUIRED', 'CompanyArgs', 'CompanyOutput', 5000, 'ACTIVE'),
  -- one row per service in §B7.3; a service with no provider message stays SUSPENDED
  ('auth.login',  'POST', '/api/v1/auth/login',    'SINGLE', 'market.auth.request.v1',
   'ARG', 'user', 'NONE', 'LoginArgs', 'LoginOutput', 10000, 'ACTIVE');

INSERT INTO mds.gw_exchange_map (provider, global_exchange, wire_exchange) VALUES
  ('TTL', 'XHKG', 'HK'), ('TTL', 'XSHG', 'SH'), ('TTL', 'XSHE', 'SZ'), ('TTL', 'XNYS', 'XNYS');

INSERT INTO mds.gw_symbol_rule (provider, exchange, rule) VALUES
  ('TTL', 'XHKG', 'STRIP_ZERO'), ('TTL', 'XSHG', 'PAD6'), ('TTL', 'XSHE', 'PAD6'), ('TTL', 'XNYS', 'PASSTHROUGH');
```

**Forward Handler**

```sql
CREATE TABLE mds.gw_operation_config (
    service_code      varchar(80)  NOT NULL,     -- = gateway_route.service_code: chart, auth.login
    provider          varchar(50)  NOT NULL,     -- TTL — the provider this deployment is configured for
    msg_type          varchar(60)  NOT NULL,     -- HISTORICAL_DATA | LOGIN
    channel           varchar(20)  NOT NULL,     -- info | snapshot
    needs_session     boolean      NOT NULL DEFAULT true,     -- false for a login: it creates the session
    request_jsonata   text         NOT NULL,
    response_jsonata  text         NOT NULL,
    multipart_mode    varchar(20)  NOT NULL DEFAULT 'single', -- single | appendPath | collect
    multipart_path    varchar(120),              -- for appendPath, e.g. points.data
    timeout_ms        integer      NOT NULL DEFAULT 5000,
    bulkhead_group    varchar(30)  NOT NULL DEFAULT 'default', -- market-data | reference | auth
    active            boolean      NOT NULL DEFAULT true,
    version           bigint       NOT NULL DEFAULT 1,
    updated_at        timestamptz  NOT NULL DEFAULT now(),
    PRIMARY KEY (service_code, provider)
);

CREATE TABLE mds.gw_exchange_map (
    provider          varchar(50)  NOT NULL,
    global_exchange   varchar(10)  NOT NULL,     -- XHKG
    wire_exchange     varchar(10)  NOT NULL,     -- HK
    active            boolean      NOT NULL DEFAULT true,
    PRIMARY KEY (provider, global_exchange)
);

CREATE TABLE mds.gw_symbol_rule (
    provider          varchar(50)  NOT NULL,
    exchange          varchar(10)  NOT NULL,     -- global exchange
    rule              varchar(20)  NOT NULL,     -- STRIP_ZERO | PAD6 | PASSTHROUGH
    active            boolean      NOT NULL DEFAULT true,
    PRIMARY KEY (provider, exchange)
);

CREATE TABLE mds.gw_enablement (
    region            varchar(10)  NOT NULL,
    exchange          varchar(10)  NOT NULL,
    service_code      varchar(80)  NOT NULL,
    active            boolean      NOT NULL DEFAULT true,
    PRIMARY KEY (region, exchange, service_code)
);
```

**The `auth.login` row** — the FORWARD `serviceCode` of §C1

```sql
INSERT INTO mds.gw_operation_config
  (service_code, provider, msg_type, channel, needs_session,
   request_jsonata, response_jsonata, multipart_mode, timeout_ms, bulkhead_group)
VALUES ('auth.login', 'TTL', 'LOGIN', 'info', false, $req$
{ "id": args.user, "msgType": "LOGIN",
  "data": { "device": provider.dataDevice, "Agreement": provider.agreement,
            "language": provider.language, "entity": provider.entity,
            "password": $providerSecret("mds.password"),
            "token": $loginToken(args.user),
            "email": $providerSecret("mds.email") },
  "device": provider.device, "version": provider.version,
  "reqTime": $millis(), "entity": provider.entity }
$req$, $res$
{ "mdsToken": $exists(sessionID) ? sessionID : detail.data,
  "user": id,
  "services": $mapWireServices(service) }
$res$, 'single', 10000, 'auth');
```

`reqId` is added by the socket, and `sessID` by the pipeline when `needs_session` is true. The JSONata text never contains a secret — only the helper functions registered in code can reach one (§C7.6).

## D2. Kafka Topics and Envelopes — **Drafted**

Kafka is the only transport between MDS services, so this section is a contract every service obeys. It ships as a shared library; a service that hand-rolls its own envelope is a defect.

### D2.1 One envelope, everywhere — **Settled**

![One envelope for every service](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/gwd-05-envelope.png)

**Text alternative:** A request and a reply side by side. Both carry a header — message id, message type, schema version, correlation and causation ids, the time it occurred, and on a request an absolute deadline and the `replyTo` topic and partition. A request's security block carries the provider token in FORWARD or the hashed customer reference in STORAGE; a reply carries no credential. The payload names the `serviceCode`, the item index for a batch item, the request id for a single call and the `args`; a reply's payload carries `isSuccess` with either `data` or an error of code, message, retryable and details. Four rules hold everywhere: one request one reply, a request past its deadline is dropped, unknown fields are ignored, and the security block is never logged.

```
Envelope
├─ header      identical on every topic, readable without deserializing payload
├─ security    present only on messages carrying an authorized action
└─ payload     shape selected by header.messageType
```

**The envelope carries no business fields.** `serviceCode`, `symbol`, `exchange`, `args` all live in `payload`. Adding a message type never changes the envelope.

| Gain | Cost |
|---|---|
| Any service routes, logs, traces and dead-letters a message without knowing its payload | Payload is opaque, so filtering on a business field needs the value promoted into a header or the key |
| One serializer, one set of tests, one library | Envelope changes are breaking for everyone, so they must be rare |

### D2.2 Header

| Field | | Purpose |
|---|---|---|
| `messageId` | Essential | Unique per message. **The idempotency key** — a consumer that has processed it may skip |
| `messageType` | Essential | Selects the payload schema — `ServiceRequest` and `ServiceReply` carry every request and reply; an event declares its own |
| `schemaVersion` | Essential | Minor version of that payload schema |
| `correlationId` | Essential | Constant across one whole interaction — a batch, a session, an admin change |
| `causationId` | Essential | The `messageId` that caused this one. Gives a full causal chain when debugging |
| `occurredAt` | Essential | UTC instant the producer created the message |
| `source` | Essential | Producing service and instance |
| `deadlineAt` | Essential on requests | Absolute. A consumer past it abandons rather than executes |
| `replyTo` | Essential on requests | Reply routing address — see D2.7 |
| `traceId` | Optional | Distributed tracing correlation |

### D2.3 Security block

Present only on a message representing an action already authorized at the gateway.

| Field | | Purpose |
|---|---|---|
| `subjectRef` | Essential in STORAGE | **Hashed** customer reference — never a raw customer id, never a token |
| `scope` | Essential in STORAGE | The offerings authorized for this action. Handlers execute it and never re-derive it |
| `providerToken` | Forward topic only | The customer's provider token, as received (§B1.5). Never logged, never copied to a dead-letter record, a retry diagnostic or an error |
| `decisionRef` | Optional | Which validation decision granted the scope, for audit reconstruction |

**No token enters a Kafka message, with one exception:** `providerToken` on `market.forward.request.v1`. It never reaches a dead-letter topic, a retry diagnostic or a log; the topic keeps it for one minute (§B7.5).

<mark>Forwarding a message through Kafka is itself a form of storage.</mark>

### D2.4 Topic naming — **Settled**

```
market.<domain>.<name>[.<role>].v<major>
```

| Segment | Values |
|---|---|
| `domain` | on a request topic, the service that owns it — `forward` · `query` · `auth` · `package`, never a verb. Otherwise the subject — `instrument` · `gateway` · `config` · `stream` · `billing` · `admin` |
| `name` | The subject within that domain |
| `role` | `request` · `reply` · `command` · `outcome` — omitted for event streams |
| `v<major>` | Payload major version. **A breaking change is a new topic, never a mutated one** |

### D2.5 Topic catalogue

**Config — compacted, infinite retention, latest state per key**

| Topic | Key | Producer → Consumer |
|---|---|---|
| `market.config.gateway-route.v1` | `serviceCode` | Admin → Gateway (§C2.2) |
| `market.config.forward-mapping.v1` | `serviceCode:provider` | Admin → Forward Handler (§D1.19) |
| `market.config.offering.v1` | `offeringCode` | Admin → Route & Offering |
| `market.config.provider.v1` | `providerCode` | Admin → Ingestion, Forward |

**Request and reply — short retention, nothing replays a request**

| Topic | Key | Consumer group |
|---|---|---|
| `market.forward.request.v1` | `correlationId:itemIndex` for a batch item · the row's key strategy for a single call | `mds.forward` |
| `market.query.snapshot.request.v1` | `correlationId:itemIndex` | `mds.query.snapshot` |
| `market.query.timeseries.request.v1` | `correlationId:itemIndex` | `mds.query.timeseries` |
| `market.query.reference.request.v1` | `correlationId:itemIndex` | `mds.query.reference` |
| `market.auth.request.v1` | The user — STORAGE deployments | `mds.auth` |
| `market.package.request.v1` | `subjectRef` — STORAGE deployments | `mds.package` |
| `market.gateway.reply.v1` | The partition named in `replyTo` | Client API Gateway — static partitions |

**Market data event streams — retention matched to the reprocessing window**

| Topic | Key |
|---|---|
| `market.instrument.trade.v1` | `exchange:symbol` |
| `market.instrument.quote.v1` | `exchange:symbol` |
| `market.instrument.bar.v2` | `exchange:symbol` |
| `market.instrument.reference.v1` | `exchange:symbol` |
| `market.instrument.financial.v1` | `exchange:symbol` |
| `market.instrument.universe.v1` | `exchange` |

**Commerce — audit-grade, long retention**

| Topic | Key | Direction |
|---|---|---|
| `market.billing.charge.request.v1` | `subscriptionRef` | MDS → trading core |
| `market.billing.charge.outcome.v1` | `subscriptionRef` | trading core → MDS |
| `market.subscription.changed.v1` | `subjectRef` | Subscription → Gateway |

**Streaming and admin**

| Topic | Key | Note |
|---|---|---|
| `market.stream.command.v1` | `sessionKey` | A session's commands — `sub`, `unsub`, `stream.close` — and each instance's heartbeat (§B4.5) |
| `market.stream.delivery.v1` | `sessionKey` | One session's frames, carrying its `connectionId` (§C10.3) |
| `market.admin.command.v1` | `entityRef` | |
| `market.admin.outcome.v1` | `correlationId` | |

### D2.6 Key strategy — **Settled**

> **The key answers one question: what must stay ordered relative to what?** Everything else is a consequence.

| Intent | Key | Why |
|---|---|---|
| Per-instrument ordering — **event streams only** | `exchange:symbol` | All events for one instrument reach one partition, so ordering holds. Applies to `market.instrument.*`, never to request topics |
| Independent requests | `correlationId:itemIndex` | Reads need no ordering relative to each other. Spreads evenly, and is well defined for an item carrying many symbols, no symbol, or several exchanges |
| One user's calls in order | `args.{key_arg}` — strategy `ARG` | A login keys on the user, so one user's logins do not overtake each other |
| One customer's calls in order | `subjectRef` — strategy `SUBJECT` | A subscribe and a cancel for the same customer must arrive in order |
| One session's stream | `sessionKey` — strategy `SESSION_KEY`, `xxHash64(salt ‖ sessionID)` | Every message of a session lands on the partition its `market-stream` instance owns, and its commands reach the `forward-service` instance holding the upstream session (§B4.2) |
| Reply routing | `replyTo` | Delivers to the partition the waiting instance owns |
| Latest state per entity | entity id | Compaction keeps exactly the current value |
| Even spread, order irrelevant | `correlationId` | Batch items may land on different partitions — they are independent by design |
| Upstream session affinity | — | Not expressed in the key. The Forward Handler pools sessions per provider internally; encoding the provider in the partition key would create a hot partition whenever one provider dominates traffic |

**Never key on a low-cardinality value alone.** Keying market data on `exchange` would put every XHKG instrument on one partition. The symbol must be in the key.

### D2.7 Reply routing — **Settled**

Gateway instances use **static partition assignment** on reply topics: an instance owns a fixed partition derived from its ordinal, and puts that partition in `replyTo`.

| Gain | Cost |
|---|---|
| No rebalance, so an in-flight request is never orphaned by a consumer-group reassignment | Partition count must be at least the maximum gateway instance count |
| No read amplification — an instance reads only its own replies | Scaling past the partition count needs a partition increase |
| `replyTo` is stable and predictable, which makes tracing simple | |

### D2.8 Retention and cleanup

| Class | Cleanup | Retention | Reason |
|---|---|---|---|
| Config | compact | infinite | A restarting service must rebuild current state from the topic |
| Query request — STORAGE | delete | ~1 hour | High volume, and nothing ever replays a request |
| Forward request · gateway reply | delete | 1 minute | Carry a token or a session id (§B7.5, §B11.6) |
| Market data streams | delete | reprocessing window | Long enough to rebuild derived data after a defect |
| Commerce | compact + long delete | audit period | Evidence for licence declarations and payment reconciliation |
| Dead letter | delete | 30 days | Long enough to investigate |

### D2.9 Consumer groups

| Case | Convention |
|---|---|
| A service consuming work | One group per service — `mds.<service>`. Scale by adding instances |
| Every instance needs every record (config) | One group **per instance**, so no instance misses a snapshot |
| Reply topics | Static assignment, not group subscription — see D2.7 |

### D2.10 Error envelope and codes — **Settled**

| Field | | Purpose |
|---|---|---|
| `code` | Essential | Stable and machine-readable. Part of the public contract |
| `message` | Essential | Human-readable. **Never contains a provider payload, a token, or a secret** |
| `retryable` | Essential | Whether the caller may retry — removes guesswork from clients and from internal retry logic |
| `details` | Optional | Structured and code-specific, e.g. `missingOffering` |

| Prefix | Meaning | Retryable |
|---|---|---|
| `REQUEST_*` | The caller's request is malformed or invalid | Never |
| `AUTH_*` | Token or identity problem | Only after obtaining a new token |
| `ENTITLEMENT_*` | Authorized identity, insufficient rights | Never without a subscription change |
| `UPSTREAM_*` | Provider-side failure | Usually |
| `STORAGE_*` | MDS store failure | Usually |
| `TIMEOUT_*` | Deadline exceeded | Usually |
| `INTERNAL_*` | Anything else | Sometimes |

Adding a code is a minor change. **Changing what an existing code means is breaking** — clients branch on it.

### D2.11 Schema evolution — **Settled**

| Change | Treatment |
|---|---|
| Add an optional payload field | Minor. Bump `schemaVersion`, same topic |
| Add a new `messageType` | Minor. No topic change |
| Remove, rename or retype a payload field | **Major. New topic at `v+1`**, dual-publish until every consumer moves, then retire the old topic |
| Change the header | Platform-wide breaking change. Requires an explicit migration plan |

Consumers must **ignore unknown payload fields** rather than reject them — without this rule no additive change is safe.

### D2.12 Dead-letter policy

A message that cannot be processed goes to `market.<service>.dlq.v1` with its **header intact and payload redacted per policy**. The header alone — `messageType`, `correlationId`, `causationId`, `source` — is enough to locate the original interaction.

No token, no provider payload, and no licensed market data content is written to a dead-letter topic. The `providerToken` field is removed before a forward request is dead-lettered.

### D2.13 What the shared library provides

![What the shared package does](https://raw.githubusercontent.com/tuanha21/GTP-MDS-docs/main/designs/assets/market-data-server/gwd-08-shared-package.png)

**Text alternative:** On the caller's side a developer builds a service call — topic, key, `serviceCode`, args, deadline and security — and hands it to `RequestClient`, which builds the envelope and key, publishes, waits on its own reply partition, matches the reply by `causationId` and bounds the calls in flight. The request travels on the request topic to the receiving service, where `ServiceRouter` consumes it, drops it if it is past its deadline, dispatches by `serviceCode`, replies exactly once and turns an exception into an error; the developer there writes only the annotated handler that returns a result or throws. The reply returns on `market.gateway.reply.v1`. The package also holds the key strategies, the error codes with their retryability, redaction, the dead-letter publisher, the bridge to the trading core's message format, and Spring Boot auto-configuration.

Lives in `libs/common-java` as `com.grooo.common.messaging`, alongside the existing shared code.

| Component | Contents |
|---|---|
| Envelope | `ServiceRequest`, `ServiceReply`, `ServiceError`; serializer; header validation |
| Keys | The key strategies of §D2.6, so a producer cannot invent its own |
| `RequestClient` · `BatchCall` | Caller side: publish, wait on the instance's own partition, match by `causationId`, enforce the deadline, bound the calls in flight, assemble a batch by `itemIndex` |
| `ServiceRouter` · `@ServiceHandler` | Receiver side: consume, drop expired requests, dispatch by `serviceCode`, reply exactly once, map an exception to an error |
| Errors | The code catalogue of §D2.10 and its retryability rules |
| Redaction · dead letter | Mask the security block in logs and traces; strip it before publishing to `market.<service>.dlq.v1` |
| Core bridge | Converts to and from the trading core's own message format |
| Spring Boot auto-configuration | `grooo.messaging.*` properties produce a ready `RequestClient` or `ServiceRouter` |

A service that hand-rolls its own envelope, key or reply is a defect.

### D2.14 Reconciling with topics already running

Three naming conventions exist in the codebase today. The target is D2.4; the table below is the migration.

| Existing | Target | Change |
|---|---|---|
| `market.instrument.*.v1` / `bar.v2` | unchanged | Already conforms |
| `market.admin.ttl.command.v1` / `.reply.v1` | unchanged | Already conforms |
| `market.config.snapshot.v1` | `market.config.provider.v1` | Name the subject, not the mechanism |
| `market.api-route-config.v1` | `market.config.gateway-route.v1` | One row per `serviceCode` (§D1.19); move into the `config` domain; drop kebab-case |
| `market.processor.dlq` | `market.processor.dlq.v1` | Add the version segment |
| `market.request.storage.v1` | `market.query.{snapshot,timeseries,reference}.request.v1`, and one topic per further business — `market.auth.request.v1`, `market.package.request.v1` | **One topic per business service** (§B3.6) — one shared STORAGE topic forces either mis-delivery or read amplification |
| `market.request.forward.v1` | `market.forward.request.v1` | Align with the convention |
| `market.reply.v1` | `market.gateway.reply.v1` | Name the reply topic after the service that waits on it (§B11.4) |

Migration is dual-publish, move consumers, then retire — the same rule as any major version change.

## D3. Security — **Open**

Blocked on: key management for payload encryption.

### D3.1 Entitlement denial — **Settled**

A request lacking entitlement is **blocked**, with an error code identifying the missing offering. MDS never silently downgrades. A subscription prompt is a client-side experience built on that code; MDS supports it by exposing the customer's current entitlements. Applies to STORAGE deployments; in FORWARD the provider's refusal is returned as `UPSTREAM_*`.

### D3.2 Tokens — **Drafted**

MDS issues no token, stores none, and in FORWARD validates none (§A5). Rules: storage §A5.3 · pass-through §B1.4 · the forwarded token §B1.5 · reading a JWE later §C3.

## D4. Reliability — **Drafted**

### D4.1 Availability of the access path

| Situation | Behaviour |
|---|---|
| Login Server or TTL login unreachable | New logins fail. Tokens already issued keep working until they expire |
| STORAGE — entitlement state unavailable, new protected requests and subscriptions | Rejected immediately; MDS fails closed |
| STORAGE — existing streams | Bounded grace of up to 60 seconds while MDS recovers; terminated with an explicit reason code if entitlement state cannot be re-established |

Delivering realtime data to a customer whose entitlement cannot be verified is a licensing exposure, so the window is short and explicitly bounded.

### D4.2 Latency budget — **Open (D13)**

| | p95 | p99 |
|---|---|---|
| One Kafka round trip | 3–6 ms | 8–15 ms |
| Two round trips | 6–12 ms | 16–30 ms |

Engineering estimates. There is no token check on the request path, so a query is one round trip. Measured figures to be published per hop after performance testing.

### D4.3 Response consistency across modes — **Settled**

The public DTO is identical for STORAGE and FORWARD; mode and freshness appear in metadata.

Identical shape does not guarantee identical **field completeness**. Where one provider does not supply a field another does, the field is present but empty. A per-service field-availability matrix will be published.

Each deployment runs one mode (§A8), so a client never sees the two completeness profiles mixed within one deployment.

### D4.4 Physical design — **Drafted**

Logical structure is §D1. This section covers what makes it survive production load.

#### Indexes the current schema is missing

PostgreSQL does not index foreign keys automatically. Every entry below serves a query the design depends on.

| Index | Serves | Why it is missing today |
|---|---|---|
| `instrument_provider_mapping (provider_symbol)` | Tick resolution — provider symbol to instrument | The only index leads with `provider_code`, so a lookup by symbol alone cannot use it. **This is a sequential scan on the hottest reference lookup in ingestion** |
| `instrument (exchange, status)` | Symbol universe enumeration | — |
| `instrument (isin)` where not null | Cross-venue and cross-listing lookup | — |
| `instrument (underlying_instrument_id)` | Option and warrant chains by underlying | Unindexed foreign key |
| `instrument_classification (node_id)` | Screener — sector to constituents | Key leads with instrument, so the reverse direction is unindexed |
| `instrument_index_constituent (constituent_instrument_id)` | Which indices contain this instrument | Same reverse-direction problem |
| `corporate_action (instrument_id, ex_date desc)` | Adjustment factor chain at query time | Unindexed foreign key on a path every historical read uses |
| `instrument_trading_halt (instrument_id, halt_start desc)` | Current halt state | Unindexed foreign key |
| `instrument_daily_attribute_value (attribute_code, trade_date desc)` | Cross-sectional daily values | — |
| `market_ranking_snapshot (region, category, ranking_type, snapshot_time desc)` | Ranking serve path | — |

All are created with `CREATE INDEX CONCURRENTLY` in a migration marked non-transactional, so a deploy never takes a write lock on a live table.

#### Partition maintenance

| Table | Current coverage | Risk |
|---|---|---|
| `ohlc_bar` | Full calendar year, per resolution | — |
| `market_trade_raw`, `market_quote_raw`, `index_breadth_bar` | **Three months only** | Inserts fail with no partition once the window is passed. This is a hard outage, not a degradation |

Partition creation must have an owner: either a scheduled maintenance job or `pg_partman`. A default partition may be kept as a safety net, but attaching new partitions while one exists forces a scan of it, so it is a last resort rather than the mechanism.

#### Write-path characteristics

| Table | Characteristic | Mitigation |
|---|---|---|
| `instrument_snapshot` | Upsert per instrument every 1–3s produces continuous row-version churn; the order-book payload is large enough to be stored out of line, so each update rewrites it | Lower fill factor to allow in-page updates, per-table aggressive autovacuum, and consider holding the order book only in Redis — it is stale within seconds and is the most expensive column |
| `market_trade_raw`, `market_quote_raw` | A surrogate identity key on a partitioned table draws from one global sequence, and the dedup unique key already identifies the row | Drop the surrogate key and make the dedup key primary — removes a column, an index, and a sequence contention point |
| `market_ranking_snapshot` | Unpartitioned, unindexed, with a growing payload per snapshot | Partition by snapshot time and apply retention |

#### Store separation

Tick ingestion and reference serving are opposite workloads — sustained sequential writes against low-latency random reads — competing for the same buffers, WAL and autovacuum workers.

| Step | Effect |
|---|---|
| Separate instances for reference and time-series | Workload isolation, independent tuning |
| Compression on the time-series store | Order-of-magnitude storage reduction and faster range scans |

Sizing targets for both are pending the concurrency figures in **D13**.

## D5. Licence Reporting — **Drafted**

| Layer | Content | Cardinality |
|---|---|---|
| **Subscription state** | Which customers held which package and offerings, over what period | Per customer per package |
| **Daily access summary** | Realtime data was actually accessed | One row per customer / exchange / depth / timing / date |
| **Monthly declaration** | Aggregated per exchange and month, split by customer classification. Produced by the job in §B9 | One report per exchange per month |

**Volume.** Ceiling ~15M rows/month at 100,000 customers across five exchanges; realistic closer to 2M. Both retain comfortably for seven years with monthly partitioning.

**Scope.** STORAGE deployments only. In a FORWARD deployment each call reaches the provider under the customer's own token, so the provider holds the per-customer record and MDS keeps none.

## D6. Technology Baseline — **Settled**

| Area | Baseline |
|---|---|
| Runtime / framework | Java 21, Spring Boot 3.3 |
| Public edge | REST/JSON with OpenAPI; WebSocket for streaming |
| Backbone | Kafka |
| Configuration, reference and commerce store | PostgreSQL |
| Historical analytics | ClickHouse |
| Cache and hot state | Redis |
| Provider integration | Adapter modules over REST, WebSocket, TCP or vendor protocol |

## D7. Open Decisions

| # | Decision | Owner | Blocks |
|---|---|---|---|
| D1 | Offering scope by exchange or by market | Architecture | §C5, offering code format, token size |
| D2 | Offering code format when depth does not apply | Architecture | §C5 |
| D3 | Entitlement policy for reference and fundamental endpoints | Product | §C6, §B3 |
| D4 | Anonymous access to basic or delayed data | Product / Legal | §B3 |
| ~~D5~~ | ~~Reply routing across gateway instances~~ — **resolved**: static partition assignment, key `replyTo` | — | §D2.7 |
| ~~D6~~ | ~~Streaming fan-out mechanism~~ — **resolved**: Kafka partitioning on the session key; no socket cluster | — | §C10.3, §B4.2 |
| D7 | Admin identity source | Architecture | §C11 |
| D8 | Payment-failure access policy — default grace period | **Customer / Product** | §A6.4 |
| D9 | Cross-listed security tier | Architecture | §D1.7 |
| ~~D10~~ | ~~TTL metering model~~ — **resolved**: a forwarded call carries the customer's own TTL token | — | §C7.1 |
| D11 | TTL upstream sharing permitted | **TTL contract** | §C7.2, streaming cost model |
| D12 | TTL caching and logging limits | **TTL contract** | §D5 |
| D13 | Target concurrency and performance targets | **Customer** | §C10.1, §D4.2 |
| ~~D14~~ | ~~Route-to-handler topic shape and naming~~ — **resolved**: one topic per business service, convention in §D2.4 | — | §D2.5 |
| D15 | Ingestion redundancy — active/standby or active/active per provider | Architecture / provider contract | §C8.11 |
| D16 | Counting basis for the licence declaration — entitlement, access, or period-end | **Customer / Legal**, per exchange | §B9.3 |
| D17 | Per-exchange declaration format and submission channel | **BA / exchange contracts** | §B9.6 |
| D18 | Login Server token contract — JWE, the secret key to decrypt it, which headers the client sends | **HK Login Server team** — later | §A5.2, §C3 |
| ~~D19~~ | ~~Provider token at rest on the forward topic~~ — **accepted**: one-minute retention, masked everywhere. <mark>Forwarding through Kafka is itself a form of storage.</mark> | — | §B7.5 |
| ~~D20~~ | ~~How the login relay trusts the mapped TTL MDS user~~ — **accepted**: taken as sent. <mark>With no password on the TTL MDS login, anyone able to call the relay can obtain a session for any user id.</mark> | — | §C1.2 |
| D21 | Detecting a dead provider session while a client is only watching — probe interval, idle cutoff, and which provider message is cheapest to probe with | Architecture / **TTL contract** | §B4.4 |
| D22 | Whether MDS may log the customer in again by itself when their session expires | **Product** — TTL allows about one session per user, so it would end the customer's other device | §B4.4 |

---

## Diagram inventory

| Diagram | Status |
|---|---|
| `mds-platform-architecture` | Current (§A3) — layered overview |
| `mds-erd-00` … `mds-erd-10` | Current (§D1) — eleven ERD figures; `mds-erd-10` is §D1.19 |
| `mds-a5-login-entrance` | Current (§A5.1) — the Login Server as the entrance |
| `mds-b1-interim-login` | Current (§B1.3) — two TTL tokens |
| `mds-b10-layer-context` | Current (§B10.1) — layer context, adapter layer highlighted |
| `mds-b10-adapter-layer`, `mds-b10-event-path` | Current (§B10.5, §B10.8) |
| `mds-c8-provider-lifecycle` | Current (§C8.5) |
| `mds-access-modes` | Current (§A7) |
| `mds-market-connection-modes` | Redraw — one mode per deployment, not per market |
| `mds-component-architecture` | Redraw — query split, login relay, subscription |
| `mds-dual-path-runtime` | Redraw — show `md:rt:*` and `md:dl:*` |
| Package, subscription and entitlement | **New** — §A6 / §B2 |
| `mds-b3-query-overview` | Current (§B3.1) — non-technical overview |
| `mds-common-query-flow` | Current (§B3.3) |
| Streaming subscription flow | **New** — §B4 |
| Delayed delivery mechanism | **New** — §A9 / §B5 |
| `mds-b7-forward-flow` | Current (§B7.1) |
| `gwd-01-two-modes` | Current (§B11.1) — the mode decides the topic |
| `gwd-02-path-to-topic`, `gwd-03-route-table`, `gwd-04-config-source` | Current (§C2.1, §C2.2) |
| `gwd-05-envelope`, `gwd-08-shared-package` | Current (§D2.1, §D2.13) |
| `gwd-06-batch-assembly`, `gwd-07-reply-partitions` | Current (§C2.4) |
| `mds-b4-session-key`, `mds-b4-session-lifecycle` | Current (§B4.1, §B4.2) |
| `mds-b4-connect`, `mds-b4-expiry` | Current (§B4.3, §B4.4) |
| `mds-b4-lifecycle` | Current (§B4.5) — the connection's life and its teardown |
| `mds-c10-transports` | Current (§B4.8, §C10.4) |
| Query tiering | **New** — §C6 |
| `mds-b9-licence-declaration` | Current (§B9) |
