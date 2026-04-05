# E-Commerce Microservice Architecture — Design Diagram

This repository contains a [draw.io](https://app.diagrams.net/) architecture diagram (`MicroserviceDesignDiagram.drawio`) that illustrates a cloud-native e-commerce platform built on a microservices pattern. The sections below explain every layer, component, and data flow shown in the diagram.

---

## Table of Contents

- [High-Level Overview](#high-level-overview)
- [Actors](#actors)
- [Front End and API Gateway](#front-end-and-api-gateway)
- [Authentication and Identity](#authentication-and-identity)
- [Core Microservices](#core-microservices)
  - [Account Service](#account-service)
  - [Catalog Service](#catalog-service)
  - [Inventory Service](#inventory-service)
  - [Order Service](#order-service)
  - [Payment Service](#payment-service)
  - [Analytics Service](#analytics-service)
- [Message Broker](#message-broker)
- [Databases and Data Stores](#databases-and-data-stores)
- [Caching and Content Delivery](#caching-and-content-delivery)
- [Search Infrastructure](#search-infrastructure)
- [Analytics, Monitoring and Reporting](#analytics-monitoring-and-reporting)
- [Key Data Flows](#key-data-flows)
- [Opening the Diagram](#opening-the-diagram)

---

## High-Level Overview

The architecture follows a standard microservices pattern for an e-commerce platform:

```
User ──▶ Front End Server ──▶ API Gateway ──▶ Microservices ──▶ Helper Services ──▶ Databases
                                   │
                              Identity Server
                                   │
                            Kafka Message Broker
                                   │
                           Analytics & Monitoring
```

The system is divided into **external-facing** components (to the left of the API Gateway) and **internal-facing** components (to the right), with the API Gateway acting as the single entry point for all client traffic.

---

## Actors

| Actor | Description |
|---|---|
| **User** | End user who opens the e-commerce portal through a browser or client application. |
| **System Administrator** | Operations/admin user who monitors the platform through live analytics dashboards and Power BI reports. |

---

## Front End and API Gateway

| Component | Role |
|---|---|
| **Front End Server** | Serves the e-commerce portal UI. The user interacts with this component, which in turn makes HTTP calls to the API Gateway. |
| **API Gateway** | Central entry point for all inbound requests. It routes traffic to the appropriate microservice, enforces authentication via the Identity Server, and connects to the Kafka Message Broker for asynchronous communication. |

---

## Authentication and Identity

| Component | Role |
|---|---|
| **Identity Server** | Handles authentication and authorization. Communicates bidirectionally with the API Gateway to validate requests. |
| **Identity Database** | Stores user credentials and identity data. The Identity Server performs authentication and authorization against this database and returns **JWT access tokens** to authorized callers. |

---

## Core Microservices

Each microservice is independently deployable, owns its data, and communicates through the API Gateway. Helper services act as an intermediate layer that encapsulates business logic and data-access concerns.

### Account Service

- Manages customer account operations (registration, profile updates).
- Delegates to the **Account Helper**, which performs upsert operations on the **Customer DB**.
- Reads customer profile data from the Customer DB.

### Catalog Service

- Manages the product catalog.
- Retrieves enriched catalog data through the **Catalog Helper**.
- The Catalog Helper sources data from:
  - **CDN Profiles** — fetches catalog blob content for fast, geographically distributed delivery.
  - **Blob Storage (Gen2)** — the origin store for catalog media and assets; the CDN pulls from here.
  - **Geographically Cached Catalog Metadata (Redis Cache)** — a distributed cache that stores catalog metadata sourced from the **Catalog Store (Index DB)**.
- The Catalog Helper also communicates with the **Inventory Helper** to include inventory availability in catalog responses.

### Inventory Service

- Manages stock levels and availability.
- Delegates to the **Inventory Helper**, which reads and writes inventory data in the **Inventory DB**.
- Supports two types of operations:
  - **Fetch Inventory Details** — reads current stock levels.
  - **Upsert Inventory** — updates inventory after restocking or order fulfillment.

### Order Service

- Manages order lifecycle (creation, tracking).
- Delegates to the **Order Helper**, which creates order records in the **Order DB**.
- Sends **order request and payment request** data downstream.

### Payment Service

- Processes financial transactions.
- Delegates to the **Payment Helper**, which:
  - Sends payment requests to the **Payment Ledger DB** for record-keeping.
  - Communicates with an external **Payment Gateway** to process the actual payment.
  - Records the payment response back in the Payment Ledger DB.

### Analytics Service

- Collects and processes platform telemetry and logs.
- Feeds data to the **Anomaly Detector** for real-time anomaly detection.
- Sends logs to **Log Cold Storage (Data Lake)** for long-term retention and batch processing.

---

## Message Broker

| Component | Role |
|---|---|
| **Kafka Message Broker** | Provides asynchronous, event-driven communication between the API Gateway and back-end services via a messaging bus. Enables decoupled, reliable message delivery across the platform. |

---

## Databases and Data Stores

Each microservice owns its dedicated database, enforcing the database-per-service pattern:

| Database | Owner Service | Purpose |
|---|---|---|
| **Identity Database** | Identity Server | User credentials, roles, and tokens |
| **Customer DB** | Account Service | Customer profiles and account data |
| **Catalog Store (Index DB)** | Catalog Service | Indexed product catalog data |
| **Search Store (Indexed DB)** | Azure Search Service | Full-text search indexes |
| **Inventory DB** | Inventory Service | Stock levels and availability |
| **Order DB** | Order Service | Order records and status |
| **Payment Ledger DB** | Payment Service | Payment transactions and ledger entries |
| **Blob Storage (Gen2)** | Catalog Service | Catalog media assets and files |
| **Log Cold Storage (Data Lake)** | Analytics Service | Long-term log and telemetry storage |

---

## Caching and Content Delivery

| Component | Role |
|---|---|
| **Geographically Cached Catalog Metadata (Redis Cache)** | Distributed cache that stores catalog metadata close to users, reducing latency. Populated from the Catalog Store (Index DB). |
| **CDN Profiles** | Content Delivery Network that serves catalog blob content (images, media) from edge locations. Pulls origin data from Blob Storage (Gen2). |

---

## Search Infrastructure

| Component | Role |
|---|---|
| **Azure Search Service** | Provides indexed search query capabilities for the e-commerce portal. Connected to the API Gateway for search requests. |
| **Search Store (Indexed DB)** | Backing indexed database that feeds the Azure Search Service with search store queries. |

---

## Analytics, Monitoring and Reporting

The platform implements both **hot-path** (real-time) and **cold-path** (batch) analytics:

| Component | Path | Description |
|---|---|---|
| **Live Analytics Dashboard** | Hot | Real-time dashboard fed directly from the API Gateway. Accessed by the System Administrator. |
| **Anomaly Detector** | Hot | AI/ML-powered anomaly detection service that identifies unusual patterns in real time. Fed by the Analytics Service. |
| **Log Cold Storage (Data Lake)** | Cold | Long-term storage for logs and telemetry data. Receives data from the Analytics Service. |
| **Data Lake Analytics** | Cold | Performs batch analysis on cold log data stored in the Data Lake. Feeds analytics reports back to the API Gateway. |
| **Power BI Report for MSR** | Cold | Business intelligence reports generated from cold-path log analytics. Accessed by the System Administrator. |

---

## Key Data Flows

1. **User Browse Flow** — User → Front End Server → (HTTP) → API Gateway → Catalog Service → Catalog Helper → Redis Cache / CDN → Blob Storage / Catalog Store
2. **Authentication Flow** — API Gateway ↔ Identity Server ↔ Identity Database → returns JWT access token
3. **Account Management Flow** — API Gateway → Account Service → Account Helper → Customer DB
4. **Search Flow** — API Gateway → Azure Search Service ← Search Store (Indexed DB)
5. **Order Flow** — API Gateway → Order Service → Order Helper → Order DB
6. **Payment Flow** — API Gateway → Payment Service → Payment Helper → Payment Gateway (external) + Payment Ledger DB
7. **Inventory Flow** — API Gateway → Inventory Service → Inventory Helper → Inventory DB
8. **Async Messaging** — API Gateway ↔ Kafka Message Broker ↔ Microservices
9. **Hot-Path Analytics** — API Gateway → Live Analytics Dashboard → System Administrator
10. **Cold-Path Analytics** — Analytics Service → Log Cold Storage → Data Lake Analytics → Power BI Report → System Administrator

---

## Opening the Diagram

You can open `MicroserviceDesignDiagram.drawio` with any of the following:

- **[draw.io (diagrams.net)](https://app.diagrams.net/)** — open the file directly in your browser.
- **VS Code** — install the [Draw.io Integration](https://marketplace.visualstudio.com/items?itemName=hediet.vscode-drawio) extension.
- **GitHub** — click the `.drawio` file in the repository; GitHub renders a preview automatically.

---

## License

See [LICENSE](LICENSE) for details.
