# MDS — Market Data Server design documents

Design documents for the Market Data Server (MDS) of the Global Trading Platform.

| Document | What it covers |
|---|---|
| [System specification](designs/market-data-server-system-specification.md) | The end-to-end specification: architecture, functional flows, services, data model, Kafka topics and envelopes, and open decisions |
| [Gateway routing and message standard](designs/gateway-routing-and-messaging-design.md) | How the Client API Gateway routes a request to Kafka, the message every service exchanges, and the shared Java package |
| [Client API (OpenAPI)](designs/market-data-client-api-v2.openapi.yaml) | The client-facing REST contract |

## Figures

Every figure lives in [`designs/assets/market-data-server/`](designs/assets/market-data-server/) and is referenced by an absolute `raw.githubusercontent.com` link, so a document can be pasted into any documentation site and its figures still load.

## Status

These are working drafts. Each section of the specification states its own status — **Settled**, **Drafted** or **Open** — and open decisions are listed in §D7.
