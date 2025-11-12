# Scalable OTA Microservice Architecture

This document outlines a robust and scalable microservice-based architecture for an Over-the-Air (OTA) firmware update system for IoT devices.

The design is built for high performance, reliability, and real-time communication, handling device registration, heartbeats, status reporting, and remote commands.

## Core Components

### Table of Contents
- Core Components
- System Architecture Diagram
- Core Data Flows
- Role of Each Data Store
- Security Architecture
- Scalability & Performance
- Monitoring & Observability
- Data Model
- Deployment Architecture
- Technology Stack & Implementation Guide
- Go Microservices Architecture
- React Dashboard Implementation
- Device Implementation Example


The system is composed of several primary components:

Devices: The IoT devices in the field. They connect, publish, and subscribe via MQTT and download firmware via HTTPs.

Dashboard (Web UI): A web application for administrators to monitor device status, manage firmware, and initiate update "campaigns."

Backend Microservices: The "brain" of the system is broken into several independent services:

Dashboard Backend: The API that the front-end Dashboard talks to.

Registration Service: Manages device identity, registration, and heartbeats.

OTA Management Service: Manages firmware versions, update campaigns, and pushes commands to devices.

Message Broker (MQTT): The real-time communication hub for devices.

Technology: MQTT (e.g., Mosquitto, EMQX, or HiveMQ).

Security: TLS encryption, client certificate authentication, and ACLs to restrict device access.

Database: The system's primary source of truth for device data, firmware, and logs.

Technology: A relational database like PostgreSQL (recommended) or MySQL.

Cache Layer (Redis): High-performance cache for device state and real-time data access.

Role: Caches device information with TTL (5 minutes) for fast lookups by all services. Provides pub/sub for real-time notifications.

Why: Enables sub-millisecond access to device state without overloading PostgreSQL.

Firmware Storage: A secure, high-availability file server to store the actual firmware binary files.

Technology: A blob storage service like Amazon S3, Google Cloud Storage, or Azure Blob Storage.

Security: Pre-signed URLs with expiration for secure downloads, firmware checksums for integrity verification.

Service Coordination (etcd): A distributed key-value store used by the backend services to coordinate and share configuration.

Role: Service Discovery (services find each other), Configuration Sharing (e.g., MQTT broker address, feature flags), and Leader Election for service instances.

Note: etcd stores service configuration only, NOT device data (which would not scale).

## System Architecture Diagram (Mermaid)

This diagram shows how the microservices interact.

flowchart TD
    subgraph "Your Cloud Infrastructure"
        direction TB
  A[Admin User] -->|"1.Manages"| B(Dashboard UI)
  B <-->|"2.GraphQL API"| BSvc[Dashboard Backend Service]

        subgraph "Backend Microservices"
            BSvc
            RegSvc[Registration Service]
            OtaSvc[OTA Management Service]
        end
        
        BSvc <-->|"3.Reads/Writes"| D[PostgreSQL Database]
        RegSvc <-->|"3.Writes"| D
        OtaSvc <-->|"3.Reads/Writes"| D

        BSvc <-->|"4.Fast Read Cache"| R[(Redis Cache)]
        RegSvc -->|"4.Updates Cache"| R
        OtaSvc <-->|"4.Fast Read Cache"| R

        BSvc <-->|"5.Service Config"| E((etcd Cluster))
        RegSvc <-->|"5.Service Config"| E
        OtaSvc <-->|"5.Service Config"| E
        
  RegSvc <-->|"6.Listens Heartbeats/Status"| F((MQTT Broker + TLS))
        OtaSvc <-->|"6.Publishes Commands"| F
        OtaSvc -->|"7.Uploads Firmware"| S[Firmware Storage S3/GCS]
    end

    subgraph "Devices in the Field"
        direction TB
        F <-->|"8.Secure Pub/Sub TLS+Certs"| G([Device 1...N])
        G -->|"9.Downloads HTTPs Pre-signed URL"| S
        G -->|"10.Local Config Web UI"| G
    end

    style B fill:#e6f7ff,stroke:#0050b3
    style BSvc fill:#f6ffed,stroke:#389e0d
    style RegSvc fill:#f6ffed,stroke:#389e0d
    style OtaSvc fill:#f6ffed,stroke:#389e0d
    style D fill:#fffbe6,stroke:#d48806
    style R fill:#fff0f6,stroke:#eb2f96
    style F fill:#fff0f6,stroke:#c41d7f
    style S fill:#f9f0ff,stroke:#531dab
    style E fill:#fff1e6,stroke:#d4380d
    style G fill:#f0f0f0,stroke:#595959


## Core Data Flows (Microservice)

### Flow 1: Device Registration & Heartbeat (Enhanced with Security & Caching)

Connect: Device connects to the MQTT Broker using TLS encryption with client certificate authentication.

Subscribe: Device subscribes to devices/12345/command (ACL enforced - can only access its own topic).

Publish Heartbeat: Device publishes to devices/12345/heartbeat.

Payload: {"serial_no": "12345", "model": "TX-100", "current_firmware": "v1.0.2", "timestamp": 1699878900}

Authenticate: MQTT Broker validates the device's client certificate and enforces ACL rules.

Process: The Registration Service (subscribed to devices/+/heartbeat) receives this message.

Validate: Registration Service validates the message payload and device identity.

Store (Primary): The Registration Service writes/updates the device record in PostgreSQL Database with last_seen timestamp.

Cache (Fast Access): The Registration Service updates Redis cache:

Key: device:12345

Value: {"serial_no": "12345", "model": "TX-100", "fw": "v1.0.2", "last_seen": 1699878900, "status": "online"}

TTL: 300 seconds (5 minutes - auto-expires if device goes offline)

Notify: Registration Service publishes to Redis Pub/Sub channel device:updates for real-time notifications to other services.

### Flow 2: Broadcast OTA Update (Enhanced with Security & Caching)

Initiate: Admin uses the Dashboard to start an update campaign.

API Call: The Dashboard sends the authenticated request (JWT token) to the Dashboard Backend Service.

Create Campaign: The Dashboard Backend Service creates a campaign record in PostgreSQL Database and notifies the OTA Management Service.

Query Devices: The OTA Management Service first checks Redis cache for online devices matching the campaign criteria (e.g., model: "TX-100", fw_version: "< v1.1.0").

If cache miss: Query PostgreSQL and update Redis cache.

Generate Secure URL: For each target device, OTA Management Service generates a pre-signed URL from S3 with expiration (e.g., 1 hour).

Payload includes firmware checksum (SHA-256) for integrity verification.

Publish: The OTA Management Service publishes a command to each device's MQTT topic (e.g., devices/12345/command).

Payload: {"command": "ota_update", "version": "v1.1.0", "url": "https://s3.my-storage.com/fw/v1.1.0.bin?signature=...", "checksum": "sha256:abc123...", "size_bytes": 5242880}

Receive: All online devices receive this command instantly via their MQTT subscription.

Log: Campaign status is updated in PostgreSQL and cached in Redis for dashboard display.

### Flow 3: Device Performs Update (Enhanced with Validation & Error Handling)

Acknowledge: The Device receives the MQTT command and publishes a status update to devices/12345/status.

Payload: {"campaign_id": "camp_789", "status": "downloading", "timestamp": 1699878905}

Process Status: The Registration Service (subscribed to devices/+/status) receives the status update.

Store: Updates PostgreSQL and Redis cache with current device status.

Download: The device downloads the firmware binary from the pre-signed S3 URL over HTTPS.

Progress updates: {"campaign_id": "camp_789", "status": "downloading", "progress": 45, "timestamp": 1699878910}

Verify Checksum: Device validates the downloaded file against the provided SHA-256 checksum.

If checksum fails: {"campaign_id": "camp_789", "status": "failed", "error": "checksum_mismatch"}

Abort and report error. Registration Service logs failure in PostgreSQL.

Install: The device reports {"campaign_id": "camp_789", "status": "installing", "timestamp": 1699878920} and begins flashing.

Reboot: Device reboots into new firmware.

Reconnect: Device connects to MQTT with TLS certificate and publishes heartbeat (Flow 1) showing new firmware version.

Payload: {"serial_no": "12345", "model": "TX-100", "current_firmware": "v1.1.0", "timestamp": 1699878950}

Success: Registration Service marks campaign as "completed" for this device in PostgreSQL and Redis.

Dashboard Update: The Dashboard Backend Service reads from Redis cache (fast) to display real-time status. If Redis key expired, falls back to PostgreSQL.

### Flow 4: Device-Side Change Notification (Enhanced)

Local Change: A user accesses the device's local Web UI and changes a setting.

Publish: The device publishes this change to devices/12345/notify.

Payload: {"setting_changed": "poll_interval", "old_value": "60s", "new_value": "300s", "timestamp": 1699879000}

Process: The Registration Service (subscribed to devices/+/notify) receives this notification.

Store: Updates the device's configuration in PostgreSQL Database.

Cache Update: Updates Redis cache with the new configuration value.

Key: device:12345:config

Value: {"poll_interval": "300s", ...other settings...}

TTL: 3600 seconds (1 hour)

Sync: The Dashboard Backend Service reads from Redis cache, so the Admin immediately sees the updated "poll_interval" value on the Dashboard.

### Flow 5: Rollback on Failure (New)

Detect Failure: Device fails to boot after firmware update (watchdog timer expires or boot failure detected).

Automatic Rollback: Device's bootloader switches to backup partition (A/B partition scheme) with previous firmware.

Reconnect: Device connects to MQTT with old firmware version.

Report Failure: Device publishes to devices/12345/status.

Payload: {"campaign_id": "camp_789", "status": "rollback", "current_firmware": "v1.0.2", "failed_firmware": "v1.1.0", "error": "boot_failure", "timestamp": 1699879100}

Process: Registration Service logs the failure in PostgreSQL.

Alert: OTA Management Service triggers an alert to admins about the failed update.

Dashboard: Shows device as "Update Failed - Rolled Back" with error details.

## Role of Each Data Store (The "Why")

This architecture uses four different "data" systems, each for a specific purpose. This separation is key to performance, scalability, and security.

PostgreSQL (The Ledger - Source of Truth):

Role: The permanent, primary source of truth for all persistent data.

Stores: Device registry, firmware metadata, update campaigns, update logs, device configurations, audit trails.

Why: Excellent for structured data, complex queries (e.g., "Show me all 'TX-100' devices running firmware < v1.1.0 that haven't updated in 30 days"), ACID transactions, and data integrity.

Access Pattern: Write-heavy for device updates, read-heavy for queries and reports.

Redis (The Cache - Fast Access Layer):

Role: High-performance cache for real-time device state and frequently accessed data.

Stores: 

Current device state (online/offline, current firmware, last seen) - TTL: 5 minutes

Device configurations - TTL: 1 hour

Active campaign status - TTL: 10 minutes

Recent device events for dashboard widgets

Why: Sub-millisecond read latency (1000x faster than PostgreSQL). Reduces database load by 80-90%. Auto-expiring keys (TTL) keep data fresh. Pub/Sub for real-time notifications between services.

Access Pattern: Read-heavy (all services query Redis first before PostgreSQL). Write-through cache (Registration Service updates both PostgreSQL and Redis).

MQTT Broker (The Post Office - Transport Layer):

Role: Real-time, bidirectional message transport between devices and backend services.

Stores: Nothing permanently (messages are ephemeral, delivered once with QoS 1 or 2, then discarded).

Security: 

TLS encryption for all connections

Client certificate authentication (X.509 certificates per device)

ACL enforcement (devices can only access their own topics: devices/{device_id}/*)

Why: Designed to handle millions of persistent, low-bandwidth connections from IoT devices. Perfect for pub/sub, commands, and status updates. Low overhead protocol ideal for constrained devices.

Access Pattern: High-frequency, small messages (heartbeats every 1-5 minutes, commands as needed).

etcd (The Coordinator - Service Configuration):

Role: Distributed configuration and coordination for backend microservices only (NOT for device data).

Stores: 

Service discovery (which instances of each service are running)

Shared configuration (MQTT broker URL, PostgreSQL connection strings, feature flags)

Leader election (which Registration Service instance is primary)

System-wide settings (max firmware size, default campaign settings)

Why: Provides high-availability and strong consistency for service configuration. Ensures all microservices can find each other and share critical configuration without hardcoding. Supports distributed locks for coordinating tasks.

Does NOT Store: Device data, device states, firmware files (etcd is not designed for high-volume, frequently-changing data).

Access Pattern: Low-frequency reads (service startup, periodic config refresh), rare writes (configuration changes).

## Security Architecture

Device Authentication & Authorization:

TLS Mutual Authentication: Each device has a unique X.509 client certificate signed by your Certificate Authority (CA).

MQTT ACL Rules: Devices can only publish to devices/{device_id}/* and subscribe to devices/{device_id}/command.

Certificate Revocation: Compromised devices can be blocked via Certificate Revocation List (CRL) or OCSP.

Firmware Integrity & Security:

Digital Signatures: All firmware binaries are signed with your private key. Devices verify signatures before installation.

Checksum Verification: SHA-256 checksums provided in OTA commands. Devices validate after download.

Pre-Signed URLs: S3 URLs expire after 1 hour and are unique per device/campaign, preventing unauthorized downloads.

Encrypted Storage: Firmware stored in S3 with server-side encryption (AES-256).

Backend Service Security:

API Authentication: Dashboard uses JWT tokens with role-based access control (RBAC).

Service-to-Service: Backend microservices communicate via TLS with mutual authentication.

Database Encryption: PostgreSQL data encrypted at rest and in transit (TLS connections).

Secrets Management: API keys, certificates, and passwords stored in HashiCorp Vault or AWS Secrets Manager (never in code or etcd).

Network Security:

All MQTT traffic: TLS 1.3

All HTTP traffic: HTTPS only

All database connections: TLS

Private subnets: Backend services in private VPC, not exposed to internet

## Scalability & Performance

Expected Load:

Devices: 100,000 - 1,000,000 devices

Heartbeat frequency: Every 5 minutes (3,333 - 33,333 messages/second average)

Concurrent OTA updates: 10,000 devices simultaneously

Dashboard users: 50-100 concurrent admins

Scaling Strategy:

MQTT Broker: Clustered deployment (3-5 nodes) with load balancing. Each node handles 100K+ connections.

Redis: Redis Cluster (3+ nodes) for horizontal scaling. Read replicas for read-heavy workloads.

PostgreSQL: Primary + read replicas (2-3). Connection pooling with PgBouncer (1000+ connections).

Backend Services: Stateless microservices, auto-scaled based on CPU/memory (Kubernetes HPA).

Firmware Storage: S3 auto-scales. Use CloudFront CDN for global distribution (reduces latency and costs).

Performance Targets:

Device status query: < 50ms (Redis cache hit)

OTA command delivery: < 1 second (MQTT pub/sub)

Dashboard API response: < 200ms (95th percentile)

Firmware download speed: > 1 MB/s (depends on device network)

## Monitoring & Observability

Metrics (Prometheus):

Device metrics: Online count, firmware version distribution, update success rate

Service metrics: Request rate, error rate, latency (RED metrics)

Infrastructure metrics: CPU, memory, disk, network usage

MQTT metrics: Connection count, message rate, queue depth

Logging (ELK Stack or Loki):

Centralized logs from all services and devices

Structured JSON logs with correlation IDs

Log levels: ERROR, WARN, INFO, DEBUG

Retention: 30 days for INFO, 90 days for ERROR

Tracing (Jaeger or OpenTelemetry):

Distributed tracing for request flows across microservices

Track OTA update flow from Dashboard → OTA Service → MQTT → Device

Alerting:

Critical: Device offline rate > 5%, OTA failure rate > 10%, service downtime

Warning: High latency (> 500ms), high error rate (> 1%), disk space < 20%

## Data Model (PostgreSQL Schema)

Devices Table:
- device_id (PK, UUID)
- serial_no (UNIQUE, STRING)
- model (STRING)
- current_firmware (STRING)
- last_seen (TIMESTAMP)
- status (ENUM: online, offline, updating)
- created_at, updated_at

Firmware Table:
- firmware_id (PK, UUID)
- version (STRING)
- model (STRING)
- file_url (STRING)
- checksum (STRING)
- size_bytes (INTEGER)
- uploaded_at (TIMESTAMP)

Campaigns Table:
- campaign_id (PK, UUID)
- name (STRING)
- firmware_id (FK)
- target_criteria (JSON: {"model": "TX-100", "fw_version": "< v1.1.0"})
- status (ENUM: pending, active, completed, cancelled)
- created_by (STRING)
- created_at, updated_at

Campaign_Devices Table:
- campaign_device_id (PK, UUID)
- campaign_id (FK)
- device_id (FK)
- status (ENUM: pending, downloading, installing, completed, failed, rollback)
- error_message (TEXT, nullable)
- started_at, completed_at (TIMESTAMP)

Device_Configs Table:
- device_id (FK)
- config_key (STRING)
- config_value (STRING)
- updated_at (TIMESTAMP)

## Deployment Architecture

Recommended: Kubernetes (K8s) on AWS/GCP/Azure

MQTT Broker: 3-node cluster, dedicated node pool

Backend Services: Auto-scaled deployments (2-10 pods per service)

PostgreSQL: Managed service (AWS RDS, GCP Cloud SQL) with multi-AZ

Redis: Managed service (AWS ElastiCache, GCP Memorystore) with cluster mode

etcd: Operator-managed (etcd-operator) 3-node cluster

Firmware Storage: S3/GCS with CloudFront/Cloud CDN

High Availability:

Multi-AZ deployment for all components

Load balancers for MQTT and HTTP traffic

Database automatic failover (< 30 seconds)

Service health checks and auto-restart

Disaster Recovery:

Daily PostgreSQL snapshots (retained 30 days)

Continuous S3 replication to backup region

etcd automated backups every 6 hours

Recovery Time Objective (RTO): < 1 hour

Recovery Point Objective (RPO): < 5 minutes

## Technology Stack & Implementation Guide

Backend: Go (Golang)

Why Go for this project:

Excellent concurrency: Goroutines are perfect for handling thousands of concurrent MQTT connections and device updates

High performance: Fast execution, low memory footprint, ideal for microservices

Built-in HTTP server: net/http package for REST APIs

Strong ecosystem: Mature libraries for MQTT, PostgreSQL, Redis, gRPC

Easy deployment: Single binary compilation, no runtime dependencies

Cross-platform: Compile for Linux, Windows, ARM (perfect for diverse deployment environments)

10.1 Use GraphQL for the Dashboard API (Go)

Why GraphQL for this project:

- Client-driven queries: Dashboard can request exactly the fields it needs (reduces bandwidth and simplifies UI code).
- Subscriptions: Built-in real-time updates for campaign progress and device status (useful for dashboard live views).
- Strong typing: Schema-first development reduces mismatches between client and server.

Recommended Go approach: use gqlgen (https://gqlgen.com) — it's schema-first, well-supported, and generates type-safe resolvers.

High-level flow:

- Define a GraphQL schema (types, queries, mutations, subscriptions).
- Run gqlgen to generate types and resolver interfaces.
- Implement resolvers to read from Redis/Postgres and publish subscription events when device updates or campaign progress occurs.
- Use WebSocket-based GraphQL subscriptions (graphql-ws / gqlgen supports it) for real-time UI.

Example minimal GraphQL schema (schema.graphqls):

```graphql
scalar Time

type Device {
  device_id: ID!
  serial_no: String!
  model: String!
  current_firmware: String!
  last_seen: Time!
  status: DeviceStatus!
}

enum DeviceStatus {
  ONLINE
  OFFLINE
  UPDATING
}

type Campaign {
  campaign_id: ID!
  name: String!
  firmware_version: String!
  status: CampaignStatus!
  progress: Int
}

enum CampaignStatus {
  PENDING
  ACTIVE
  COMPLETED
  CANCELLED
}

type Query {
  device(id: ID!): Device
  devices(model: String, status: DeviceStatus, limit: Int = 50): [Device!]!
  campaign(id: ID!): Campaign
  campaigns(status: CampaignStatus): [Campaign!]!
}

input CreateCampaignInput {
  name: String!
  firmware_id: ID!
  target_criteria: String
}

type Mutation {
  createCampaign(input: CreateCampaignInput!): Campaign!
  startCampaign(id: ID!): Boolean!
}

type Subscription {
  deviceUpdated(deviceId: ID!): Device!
  campaignProgress(campaignId: ID!): Campaign!
}
```

Go server: quickstart outline using gqlgen

1) Install gqlgen and init:

```bash
go install github.com/99designs/gqlgen@latest
gqlgen init
```

2) Generated structure will include `graph/schema.graphqls`, `graph/resolver.go`, `graph/schema.resolvers.go`.

3) Implement resolvers (simplified examples):

```go
// graph/schema.resolvers.go
package graph

import (
  "context"
  "time"

  "github.com/yourorg/ota-system/internal/database"
  "github.com/yourorg/ota-system/internal/cache"
  "github.com/yourorg/ota-system/graph/model"
)

type Resolver struct {
  DB    *database.DB
  Cache *cache.Client
}

func (r *queryResolver) Device(ctx context.Context, id string) (*model.Device, error) {
  // Try Redis cache first
  if d := r.Cache.GetDevice(ctx, id); d != nil {
    return d, nil
  }
  // Fallback to DB
  return r.DB.GetDeviceByID(ctx, id)
}

func (r *mutationResolver) StartCampaign(ctx context.Context, id string) (bool, error) {
  // Update DB state, enqueue campaign work, and publish initial campaign progress via subscription
  err := r.DB.StartCampaign(ctx, id)
  if err != nil {
    return false, err
  }
  // Publish to subscription hub (implementation detail) - example:
  go func() {
    // simulate progress events
    for p := 0; p <= 100; p += 10 {
      time.Sleep(1 * time.Second)
      r.Cache.PublishCampaignProgress(id, p)
    }
  }()
  return true, nil
}

// Subscription resolvers use channels/hubs - gqlgen supports returning (<-chan *model.Campaign, error)
```

Subscription & websocket notes:

- gqlgen supports GraphQL subscriptions over WebSocket. The server must maintain channels and publish events when device updates come in (e.g., Redis Pub/Sub messages are bridged to subscription channels).
- For production, back subscription events by Redis Pub/Sub or NATS so multiple API server instances can broadcast events consistently.

Frontend (React) with Apollo Client:

- Use `@apollo/client` for queries/mutations and `graphql-ws` or `subscriptions-transport-ws` for subscriptions.
- Example usage (simplified):

```ts
// apollo.ts
import { ApolloClient, InMemoryCache, split, HttpLink } from '@apollo/client';
import { GraphQLWsLink } from '@apollo/client/link/subscriptions';
import { createClient } from 'graphql-ws';
import { getMainDefinition } from '@apollo/client/utilities';

const httpLink = new HttpLink({ uri: '/graphql' });

const wsLink = new GraphQLWsLink(createClient({ url: 'ws://localhost:8080/graphql' }));

const splitLink = split(
  ({ query }) => {
    const definition = getMainDefinition(query);
    return definition.kind === 'OperationDefinition' && definition.operation === 'subscription';
  },
  wsLink,
  httpLink,
);

export const apolloClient = new ApolloClient({
  link: splitLink,
  cache: new InMemoryCache(),
});
```

Frontend subscription example:

```tsx
import { useSubscription, gql } from '@apollo/client';

const CAMPAIGN_PROGRESS = gql`
  subscription OnCampaignProgress($campaignId: ID!) {
    campaignProgress(campaignId: $campaignId) {
      campaign_id
      progress
      status
    }
  }
`;

function CampaignProgress({ campaignId }: { campaignId: string }) {
  const { data, loading } = useSubscription(CAMPAIGN_PROGRESS, { variables: { campaignId } });
  if (loading) return <div>Waiting for progress...</div>;
  return <div>Progress: {data.campaignProgress.progress}%</div>;
}
```

Security & Auth with GraphQL:

- Use standard JWT or cookie-based auth on the HTTP endpoint. For subscriptions, pass auth token during the WebSocket connection init payload and validate on the server.
- Enforce RBAC in resolver layer (check user's roles before returning/performing sensitive operations).

Operational notes:

- Caching: Use Apollo cache on frontend and Redis on backend. For heavy list queries, implement pagination and cursor-based queries.
- Rate limiting: Protect GraphQL endpoint with rate limiting middleware (nginx, API gateway) and enforce depth/complexity limits using `github.com/99designs/gqlgen/graphql/handler/extension`.
- Monitoring: Add GraphQL-specific metrics (query latency, resolver timing) to Prometheus.

Would you like me to integrate the full gqlgen example into the repo (sample Go service + Dockerfile + small React client) so you can run it locally? If yes I will add the files, setup, and a minimal test harness.

Go Libraries & Frameworks:

Web Framework: gin-gonic/gin (fast HTTP router) or fiber (Express-like API)

MQTT Client: eclipse/paho.mqtt.golang (official MQTT client library)

PostgreSQL: lib/pq or pgx (high-performance PostgreSQL driver)

Redis: go-redis/redis (feature-rich Redis client)

etcd: go.etcd.io/etcd/client/v3 (official etcd client)

Configuration: spf13/viper (configuration management)

Logging: uber-go/zap (structured, high-performance logging)

Metrics: prometheus/client_golang (Prometheus metrics)

Testing: testify (assertions) + gomock (mocking)

Frontend: React

Why React for this project:

Component-based: Reusable UI components (DeviceCard, CampaignTable, StatusBadge)

Rich ecosystem: React Query for data fetching, React Router for navigation

Real-time updates: WebSocket or Server-Sent Events for live device status

Large community: Extensive libraries and tooling

TypeScript support: Type safety for complex data structures

React Libraries & Tools:

State Management: React Query (TanStack Query) for server state, Zustand for client state

Routing: React Router v6

UI Components: Ant Design or Material-UI (pre-built dashboard components)

Charts: Recharts or Apache ECharts (firmware distribution charts, update progress)

Data Tables: TanStack Table (virtualized tables for thousands of devices)

Forms: React Hook Form + Zod (validation)

API Client: Axios or native fetch with React Query

Real-time: WebSocket API or EventSource (Server-Sent Events)

Build Tool: Vite (fast development, optimized builds)
