# Observability Stack - Architecture Guide

> Comprehensive guide to the design, data flow, and architectural decisions behind the multi-cluster Kubernetes observability platform.

## Table of Contents

1. [Overview](#overview)
2. [Architecture Principles](#architecture-principles)
3. [System Architecture](#system-architecture)
4. [Component Deep Dive](#component-deep-dive)
5. [Data Flow](#data-flow)
6. [High Availability Design](#high-availability-design)
7. [Scalability Strategy](#scalability-strategy)
8. [Security Architecture](#security-architecture)
9. [Network Architecture](#network-architecture)
10. [Design Decisions](#design-decisions)

---

## Overview

This observability platform follows a **hub-and-spoke** model where a central infrastructure cluster acts as the monitoring hub, collecting and storing metrics from multiple application clusters (dev, staging, production).

### Key Characteristics

- **Centralized Storage**: Single VictoriaMetrics cluster stores all metrics
- **Distributed Collection**: vmagent deployed in each cluster for local scraping
- **Multi-Tenancy**: Cluster and environment labels for isolation
- **High Availability**: All components run with redundancy
- **Cloud Native**: Kubernetes-native design with CRDs and operators

---

## Architecture Principles

### 1. Separation of Concerns

```
┌─────────────────────┐
│  Collection Layer   │  ← vmagent (scraping)
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│   Storage Layer     │  ← VictoriaMetrics (persistence)
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│ Visualization Layer │  ← Grafana (dashboards)
└─────────────────────┘
```

Each layer is independent and can be scaled/upgraded separately.

### 2. Single Source of Truth

All metrics from all environments flow into one VictoriaMetrics cluster, providing:
- Unified query interface
- Cross-cluster correlation
- Centralized alerting
- Consistent retention policies

### 3. Edge Intelligence

vmagent runs in each cluster to:
- Reduce network traffic (local aggregation)
- Buffer metrics during network issues
- Add cluster-specific labels
- Filter unnecessary metrics at the source

### 4. Defense in Depth

Multiple layers of redundancy:
- Component-level (replicas)
- Storage-level (replication)
- Network-level (retry mechanisms)
- Application-level (persistent queues)

---

## System Architecture

### High-Level Architecture

```
┌───────────────────────────────────────────────────────────────────────────┐
│                        Infrastructure Cluster                             │
│                    (Central Monitoring Hub - Infra Account)               │
│                                                                           │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                     VictoriaMetrics Cluster                        │  │
│  │                                                                    │  │
│  │   ┌──────────┐      ┌──────────┐      ┌──────────┐              │  │
│  │   │  VMAuth  │      │ VMInsert │      │ VMSelect │              │  │
│  │   │  (2x)    │──────▶  (2x)    │      │  (2x)    │              │  │
│  │   │          │      │          │      │          │              │  │
│  │   │ Auth &   │      │  Write   │      │  Read    │              │  │
│  │   │  Route   │      │  Path    │      │  Path    │              │  │
│  │   └─────┬────┘      └────┬─────┘      └────┬─────┘              │  │
│  │         │                │                  │                    │  │
│  │         │     ┌──────────▼──────────────────▼──────┐            │  │
│  │         │     │       VMStorage (2x)                │            │  │
│  │         │     │    Persistent Metric Storage        │            │  │
│  │         │     │     - 100Gi gp3 per replica         │            │  │
│  │         │     │     - 15 days retention             │            │  │
│  │         │     │     - Deduplication enabled         │            │  │
│  │         │     └─────────────────────────────────────┘            │  │
│  │         │                                                         │  │
│  │         │     ┌─────────────────────────────────────┐            │  │
│  │         │     │       VMAlert (2x)                  │            │  │
│  │         └─────▶  Alerting & Recording Rules         │            │  │
│  │               └─────────────────────────────────────┘            │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                      Grafana HA Setup                              │  │
│  │                                                                    │  │
│  │   ┌──────────────────┐         ┌──────────────────┐              │  │
│  │   │   Grafana (2x)   │◀────────│  Aurora PostgreSQL│              │  │
│  │   │                  │         │                  │              │  │
│  │   │  • Dashboards    │         │  • Sessions      │              │  │
│  │   │  • Users         │         │  • Dashboards    │              │  │
│  │   │  • Datasources   │         │  • Users/Orgs    │              │  │
│  │   └────────┬─────────┘         └──────────────────┘              │  │
│  │            │                                                       │  │
│  │            │                                                       │  │
│  │   ┌────────▼─────────┐                                            │  │
│  │   │   EFS Storage    │                                            │  │
│  │   │  (Shared 10Gi)   │                                            │  │
│  │   │  • Plugins       │                                            │  │
│  │   │  • Provisioning  │                                            │  │
│  │   └──────────────────┘                                            │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │            Local Metrics Collection (Infra Cluster)                │  │
│  │                                                                    │  │
│  │   ┌────────────┐   ┌────────────────┐   ┌──────────────────┐    │  │
│  │   │  vmagent   │───│ Node Exporter  │   │ Kube State       │    │  │
│  │   │  (Local)   │   │  (DaemonSet)   │   │  Metrics (1x)    │    │  │
│  │   │            │   │                │   │                  │    │  │
│  │   │ Scrapes:   │   │ • CPU/Memory   │   │ • Pod Status     │    │  │
│  │   │ • Exporters│   │ • Disk I/O     │   │ • Deployments    │    │  │
│  │   │ • K8s API  │   │ • Network      │   │ • Nodes          │    │  │
│  │   │ • cAdvisor │   │ • Filesystem   │   │ • Services       │    │  │
│  │   └──────┬─────┘   └────────────────┘   └──────────────────┘    │  │
│  │          │                                                        │  │
│  │          └────────────────────▶ Write to VMInsert                │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│                   ┌───────────────────────────────────┐                  │
│                   │      AWS ALB (Internal)           │                  │
│                   │  • victoria-metrics.infra.service │                  │
│                   │  • grafana.infra.service          │                  │
│                   └───────────────────────────────────┘                  │
└───────────────────────────────────────────────────────────────────────────┘
                                    ▲
                                    │
                    Remote Write (Authenticated via VMAuth)
                                    │
┌───────────────────────────────────┼───────────────────────────────────────┐
│                                   │                                       │
│   ┌───────────────────────────────┼─────────────────────┐                │
│   │         Dev Cluster            │                     │                │
│   │                               │                     │                │
│   │  ┌─────────────┐  ┌───────────▼──────────┐         │                │
│   │  │ Exporters   │  │      vmagent         │         │                │
│   │  │             │◀─│                      │         │                │
│   │  │ • Node      │  │ Scrapes local        │         │                │
│   │  │ • KSM       │  │ exporters and K8s    │         │                │
│   │  └─────────────┘  │                      │         │                │
│   │                   │ Remote Write URL:    │         │                │
│   │                   │ victoria-metrics     │         │                │
│   │                   │   .infra.service     │         │                │
│   │                   │                      │         │                │
│   │                   │ Labels:              │         │                │
│   │                   │ • cluster: dev       │         │                │
│   │                   │ • environment: dev   │         │                │
│   │                   └──────────────────────┘         │                │
│   └──────────────────────────────────────────────────────┘                │
│                                                                           │
│   ┌────────────────────────────────────────────────────────┐             │
│   │         Staging Cluster                                │             │
│   │  (Similar setup with staging labels)                   │             │
│   └────────────────────────────────────────────────────────┘             │
│                                                                           │
│   ┌────────────────────────────────────────────────────────┐             │
│   │         Production Cluster                             │             │
│   │  (Similar setup with production labels)                │             │
│   └────────────────────────────────────────────────────────┘             │
│                                                                           │
│                      Remote Clusters (Application Workloads)             │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## Component Deep Dive

### VictoriaMetrics Cluster

**Architecture Pattern**: Microservices with dedicated components for write, read, and storage.

```
┌────────────────────────────────────────┐
│        VictoriaMetrics Cluster         │
├────────────────────────────────────────┤
│                                        │
│  VMAuth (Entry Point)                  │
│  ├─ Authentication                     │
│  ├─ Request routing                    │
│  └─ Load balancing                     │
│         │                              │
│         ├─▶ Write Path                 │
│         │   └─▶ VMInsert               │
│         │       ├─ Metric ingestion    │
│         │       ├─ Sharding logic      │
│         │       └─▶ VMStorage          │
│         │                              │
│         └─▶ Read Path                  │
│             └─▶ VMSelect               │
│                 ├─ Query execution     │
│                 ├─ Result aggregation  │
│                 └─▶ VMStorage          │
│                                        │
│  VMStorage (Persistence Layer)         │
│  ├─ Time-series data                   │
│  ├─ Indexing                           │
│  ├─ Replication (2x)                   │
│  └─ Retention management               │
│                                        │
│  VMAlert (Alerting Engine)             │
│  ├─ Evaluates rules                    │
│  ├─ Generates alerts                   │
│  └─ Forwards to Alertmanager           │
└────────────────────────────────────────┘
```

**Why This Design?**
- **Horizontal scalability**: Each component scales independently
- **Performance**: Dedicated read/write paths avoid contention
- **Resilience**: Component failure doesn't bring down the entire system

### Grafana High Availability

**Architecture Pattern**: Active-Active with shared state.

```
┌──────────────────────────────────┐
│       Grafana HA Setup           │
├──────────────────────────────────┤
│                                  │
│  ┌─────────┐    ┌─────────┐     │
│  │Grafana 1│    │Grafana 2│     │
│  │(Active) │    │(Active) │     │
│  └────┬────┘    └────┬────┘     │
│       │              │           │
│       └──────┬───────┘           │
│              │                   │
│       ┌──────▼──────┐            │
│       │   Aurora    │            │
│       │ PostgreSQL  │            │
│       │             │            │
│       │ • Sessions  │            │
│       │ • Dashboards│            │
│       │ • Users     │            │
│       └─────────────┘            │
│              │                   │
│       ┌──────▼──────┐            │
│       │     EFS     │            │
│       │ (RWX Storage)│           │
│       │             │            │
│       │ • Plugins   │            │
│       │ • Assets    │            │
│       └─────────────┘            │
└──────────────────────────────────┘
```

**Why This Design?**
- **No single point of failure**: Both instances serve traffic
- **Shared state**: PostgreSQL ensures consistency
- **Session persistence**: Users don't lose sessions during updates
- **Plugin sharing**: EFS allows both instances to access same plugins

### vmagent Collection Pattern

**Architecture Pattern**: Edge collection with buffering.

```
┌────────────────────────────────────┐
│          vmagent                   │
├────────────────────────────────────┤
│                                    │
│  Service Discovery                 │
│  ├─ Kubernetes API                 │
│  ├─ Static configs                 │
│  └─ DNS-based discovery            │
│         │                          │
│         ▼                          │
│  Scrape Targets                    │
│  ├─ Node Exporter (every node)    │
│  ├─ Kube State Metrics             │
│  ├─ cAdvisor (kubelet)             │
│  ├─ API Server                     │
│  └─ Annotated pods                 │
│         │                          │
│         ▼                          │
│  Processing                        │
│  ├─ Label addition (cluster, env) │
│  ├─ Metric filtering               │
│  └─ Relabeling rules               │
│         │                          │
│         ▼                          │
│  Persistent Queue (10Gi PVC)       │
│  └─ Buffer during network issues   │
│         │                          │
│         ▼                          │
│  Remote Write                      │
│  ├─ Batch processing               │
│  ├─ Compression                    │
│  ├─ Retry logic                    │
│  └─ Multiple queues (parallel)    │
│         │                          │
│         ▼                          │
│  VictoriaMetrics VMInsert          │
└────────────────────────────────────┘
```

**Why This Design?**
- **Resilient**: Persistent queue survives restarts
- **Efficient**: Batching reduces network overhead
- **Flexible**: Easy to add/remove scrape targets
- **Observable**: Self-monitoring included

---

## Data Flow

### 1. Metrics Collection Flow

```
Application/System Metrics
         │
         ▼
   ┌─────────────┐
   │  Exporters  │ (Expose metrics on /metrics endpoint)
   └──────┬──────┘
          │
          ▼
   ┌─────────────┐
   │  vmagent    │ (Scrapes metrics every 30s)
   └──────┬──────┘
          │
          ▼
   ┌─────────────┐
   │Process/Label│ (Add cluster/env labels)
   └──────┬──────┘
          │
          ▼
   ┌─────────────┐
   │Persistent Q │ (Buffer if network issues)
   └──────┬──────┘
          │
          ▼
   ┌─────────────┐
   │Remote Write │ (HTTP POST with compression)
   └──────┬──────┘
          │
          ▼ (Authenticated via VMAuth)
   ┌─────────────┐
   │  VMInsert   │ (Route to appropriate VMStorage)
   └──────┬──────┘
          │
          ▼
   ┌─────────────┐
   │ VMStorage   │ (Persist to disk with indexing)
   └─────────────┘
```

### 2. Query Flow

```
User/Dashboard
      │
      ▼
┌─────────────┐
│   Grafana   │ (PromQL query)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  VMSelect   │ (Parse query, determine time range)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ VMStorage   │ (Fetch raw data from disk)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  VMSelect   │ (Aggregate, compute, filter)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Grafana   │ (Render visualization)
└──────┬──────┘
       │
       ▼
   User Browser
```

### 3. Alert Flow

```
┌─────────────┐
│  VMAlert    │ (Evaluate rules every 30s)
└──────┬──────┘
       │
       ▼ (Query VMSelect)
┌─────────────┐
│  VMSelect   │ (Execute alert query)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  VMAlert    │ (Compare with threshold)
└──────┬──────┘
       │
       ▼ (If threshold breached)
┌─────────────┐
│ Alertmanager│ (Group, route, notify)
└──────┬──────┘
       │
       ▼
Notification Channels
(Slack, PagerDuty, Email)
```

---

## High Availability Design

### Component-Level HA

Each critical component runs with multiple replicas:

| Component | Replicas | Strategy |
|-----------|----------|----------|
| VMInsert | 2 | Stateless, load balanced |
| VMSelect | 2 | Stateless, load balanced with cache |
| VMStorage | 2 | Stateful, data replicated |
| VMAuth | 2 | Stateless, load balanced |
| VMAlert | 2 | Active-Active (deduplicated) |
| Grafana | 2 | Active-Active (shared DB) |
| vmagent (infra) | 1 | Single with persistent queue |

### Storage-Level HA

```
┌──────────────────────────────────────┐
│      Storage Redundancy              │
├──────────────────────────────────────┤
│                                      │
│  VMStorage Replication               │
│  ├─ 2 replicas with same data        │
│  ├─ Auto-deduplication               │
│  └─ Survives single replica failure  │
│                                      │
│  EBS Volumes (gp3)                   │
│  ├─ AWS-managed redundancy           │
│  ├─ Automatic snapshots              │
│  └─ 99.99% durability                │
│                                      │
│  Aurora PostgreSQL                   │
│  ├─ Multi-AZ deployment              │
│  ├─ Automatic failover               │
│  └─ Read replicas available          │
│                                      │
│  EFS (Grafana)                       │
│  ├─ Multi-AZ by default              │
│  ├─ 99.99% availability              │
│  └─ Automatic replication            │
└──────────────────────────────────────┘
```

### Network-Level HA

```
┌──────────────────────────────────────┐
│     Network Redundancy               │
├──────────────────────────────────────┤
│                                      │
│  AWS ALB                             │
│  ├─ Multi-AZ load balancing          │
│  ├─ Health checks                    │
│  └─ Automatic failover               │
│                                      │
│  Kubernetes Services                 │
│  ├─ Internal load balancing          │
│  ├─ Endpoint management              │
│  └─ Self-healing                     │
│                                      │
│  Pod Anti-Affinity                   │
│  ├─ Replicas on different nodes      │
│  ├─ AZ-aware scheduling              │
│  └─ Survives node failure            │
└──────────────────────────────────────┘
```

---

## Scalability Strategy

### Vertical Scaling (Individual Components)

```yaml
# Easy resource adjustment
resources:
  requests:
    cpu: 500m → 1        # Double CPU
    memory: 1Gi → 2Gi    # Double memory
```

### Horizontal Scaling (Replica Count)

**VMInsert / VMSelect**:
```bash
# Scale write capacity
kubectl scale deployment vminsert --replicas=4

# Scale read capacity
kubectl scale deployment vmselect --replicas=4
```

**VMStorage**:
```bash
# Add storage nodes (requires data rebalancing)
kubectl scale statefulset vmstorage --replicas=4
```

### Data Scaling (Storage Capacity)

```yaml
# Expand VMStorage volumes
persistentVolume:
  size: 100Gi → 500Gi  # Expand as needed
```

### Retention Scaling

```yaml
# Adjust retention period
extraArgs:
  retentionPeriod: 15d → 30d  # Longer retention
```

### Scalability Limits

| Component | Max Recommended | Bottleneck |
|-----------|----------------|------------|
| VMInsert | 10 replicas | Network bandwidth |
| VMSelect | 10 replicas | Query complexity |
| VMStorage | 20 nodes | Disk I/O |
| Active Series | 100M+ | Memory per VMStorage |
| Ingestion Rate | 10M samples/sec | CPU per VMInsert |

---

## Security Architecture

### Authentication & Authorization

```
┌────────────────────────────────────────┐
│        Security Layers                 │
├────────────────────────────────────────┤
│                                        │
│  Layer 1: Network                      │
│  ├─ Internal ALB only                  │
│  ├─ VPC CIDR restrictions              │
│  └─ Security groups                    │
│         │                              │
│         ▼                              │
│  Layer 2: VMAuth                       │
│  ├─ Basic authentication               │
│  ├─ Per-user/cluster credentials       │
│  └─ Rate limiting                      │
│         │                              │
│         ▼                              │
│  Layer 3: RBAC                         │
│  ├─ Kubernetes RBAC                    │
│  ├─ Service accounts                   │
│  └─ ClusterRoles (read-only)           │
│         │                              │
│         ▼                              │
│  Layer 4: Pod Security                 │
│  ├─ Non-root containers                │
│  ├─ Read-only root filesystem          │
│  ├─ Dropped capabilities               │
│  └─ SeccompProfile                     │
│         │                              │
│         ▼                              │
│  Layer 5: Secrets                      │
│  ├─ External Secrets Operator          │
│  ├─ AWS Secrets Manager                │
│  └─ Encrypted at rest                  │
└────────────────────────────────────────┘
```

### Access Control Matrix

| User/System | VMInsert | VMSelect | Grafana | Exporters |
|-------------|----------|----------|---------|-----------|
| Dev Cluster vmagent | Write | - | - | - |
| Stage Cluster vmagent | Write | - | - | - |
| Prod Cluster vmagent | Write | - | - | - |
| Grafana | - | Read | Full | - |
| Ops Team | - | Read | Full | Read |
| Developers | - | - | View | - |

### Data Encryption

- **In Transit**: TLS for all remote write connections
- **At Rest**: EBS encryption, Aurora encryption, EFS encryption
- **Secrets**: AWS Secrets Manager with KMS encryption

---

## Network Architecture

### Service Mesh

```
┌──────────────────────────────────────────────┐
│         Kubernetes Networking                │
├──────────────────────────────────────────────┤
│                                              │
│  ClusterIP Services                          │
│  ├─ vminsert.monitoring.svc.cluster.local   │
│  ├─ vmselect.monitoring.svc.cluster.local   │
│  ├─ vmstorage.monitoring.svc.cluster.local  │
│  ├─ grafana.monitoring.svc.cluster.local    │
│  └─ kube-state-metrics.monitoring...        │
│                                              │
│  Headless Services                           │
│  ├─ vmstorage (StatefulSet discovery)       │
│  └─ node-exporter (DaemonSet endpoints)     │
│                                              │
│  External Services (via ALB)                 │
│  ├─ victoria-metrics.infra.service           │
│  └─ grafana.infra.service                    │
└──────────────────────────────────────────────┘
```

### Port Mapping

| Service | Port | Protocol | Purpose |
|---------|------|----------|---------|
| VMInsert | 8480 | HTTP | Metric ingestion |
| VMSelect | 8481 | HTTP | Query API |
| VMStorage | 8482 | HTTP | Storage API |
| VMAuth | 8427 | HTTP | Auth & routing |
| VMAlert | 8880 | HTTP | Alert API |
| Grafana | 3000 | HTTP | Web UI |
| vmagent | 8429 | HTTP | Scrape & self-metrics |
| Node Exporter | 9100 | HTTP | Host metrics |
| Kube State Metrics | 8080 | HTTP | K8s metrics |

---

## Design Decisions

### Why VictoriaMetrics Over Prometheus?

| Aspect | Prometheus | VictoriaMetrics | Decision |
|--------|------------|-----------------|----------|
| **Storage Efficiency** | 1x | 10x compression | ✅ VM |
| **Multi-tenancy** | Federation (complex) | Native labels | ✅ VM |
| **HA Setup** | Complex (Thanos/Cortex) | Built-in cluster | ✅ VM |
| **Query Performance** | Good | Excellent | ✅ VM |
| **Resource Usage** | High memory | Low memory | ✅ VM |
| **Ecosystem** | Mature | Growing | ⚖️ Neutral |

**Conclusion**: VictoriaMetrics provides better economics and easier operations at scale.

### Why PostgreSQL for Grafana Backend?

| Aspect | SQLite | PostgreSQL |
|--------|--------|------------|
| **Concurrency** | Limited | Excellent |
| **HA Support** | No | Yes (Aurora) |
| **Performance** | Good for single user | Excellent for multi-user |
| **Backup/Restore** | File-based | Native tools + Aurora automated |
| **Multi-instance** | No | Yes (shared state) |

**Conclusion**: PostgreSQL enables true HA and multi-user performance.

### Why Hub-and-Spoke Over Federated?

**Hub-and-Spoke** (Our choice):
```
Dev ──┐
      ├──▶ Central VM ──▶ Single Grafana
Prod ─┘
```

**Federated**:
```
Dev VM ──┐
         ├──▶ Federated Prometheus ──▶ Grafana
Prod VM ─┘
```

| Aspect | Hub-and-Spoke | Federated |
|--------|---------------|-----------|
| **Complexity** | Low | High |
| **Query Performance** | Fast (single source) | Slow (multi-query) |
| **Maintenance** | One system | Multiple systems |
| **Data Retention** | Unified | Inconsistent |
| **Cost** | Lower | Higher |

**Conclusion**: Hub-and-spoke is simpler, faster, and cheaper for our use case.

### Why EFS for Grafana Storage?

| Aspect | EBS | EFS |
|--------|-----|-----|
| **Access Mode** | ReadWriteOnce | ReadWriteMany |
| **HA Support** | No (single pod) | Yes (multiple pods) |
| **Performance** | Faster | Good enough |
| **Cost** | Lower | Slightly higher |

**Conclusion**: EFS enables true HA for Grafana with shared plugin storage.

### Why vmagent in Each Cluster?

**Alternative**: Central scraping from infra cluster

| Aspect | Local vmagent | Central Scraping |
|--------|---------------|------------------|
| **Network Traffic** | Low | High (all metrics) |
| **Latency** | Low | High |
| **Resilience** | High (buffering) | Low (network dependent) |
| **Security** | Good (cluster boundary) | Complex (cross-cluster) |
| **Flexibility** | High (per-cluster rules) | Low |

**Conclusion**: Local vmagent is more efficient, resilient, and flexible.

---

## Performance Characteristics

### Expected Metrics Volume

| Environment | Nodes | Pods | Series | Samples/sec |
|-------------|-------|------|--------|-------------|
| Dev | 3 | ~50 | ~10k | ~300 |
| Staging | 5 | ~100 | ~20k | ~600 |
| Production | 10 | ~300 | ~50k | ~1,500 |
| **Total** | **18** | **~450** | **~80k** | **~2,400** |

### Resource Requirements (Actual)

Based on above metrics volume:

| Component | CPU | Memory | Storage | Comments |
|-----------|-----|--------|---------|----------|
| VMInsert | 500m | 1Gi | - | Handles ~5k samples/sec |
| VMSelect | 500m | 2Gi | 50Gi cache | Query load dependent |
| VMStorage | 1 core | 4Gi | 100Gi | 15 days @ 2.4k samples/sec |
| VMAuth | 100m | 128Mi | - | Lightweight proxy |
| Grafana | 500m | 1Gi | 10Gi | 10-20 concurrent users |
| vmagent | 250m | 512Mi | 10Gi | Per cluster |

### Query Performance

- **Simple queries** (up, node_cpu): <100ms
- **Complex aggregations**: 100-500ms
- **Dashboard load** (10 panels): 1-2s
- **95th percentile latency**: <1s

---

## Future Enhancements

### Planned Improvements

1. **Long-term Storage**
   - S3 backup for historical data
   - Tiered storage (hot/cold)

2. **Advanced Alerting**
   - Alertmanager integration
   - Custom notification channels
   - Alert silencing and grouping

3. **Distributed Tracing**
   - Tempo integration
   - Trace-to-metrics correlation

4. **Log Aggregation**
   - Loki integration
   - Unified observability (metrics + logs + traces)

5. **Auto-scaling**
   - VPA for component right-sizing
   - HPA for query load

6. **Multi-region**
   - Cross-region replication
   - Disaster recovery

---

## Conclusion

This architecture provides a robust, scalable, and cost-effective observability platform suitable for production workloads. The design decisions prioritize:

- **Operational simplicity** over complexity
- **Cost efficiency** over feature abundance
- **Reliability** over bleeding-edge features
- **Maintainability** over perfect optimization

The hub-and-spoke model with VictoriaMetrics at the core provides the right balance of centralization and resilience for multi-cluster Kubernetes monitoring.

---

## References

- [VictoriaMetrics Architecture](https://docs.victoriametrics.com/Cluster-VictoriaMetrics.html)
- [Grafana HA Setup](https://grafana.com/docs/grafana/latest/setup-grafana/set-up-for-high-availability/)
- [Prometheus Naming Conventions](https://prometheus.io/docs/practices/naming/)
- [Kubernetes Monitoring Best Practices](https://kubernetes.io/docs/tasks/debug-application-cluster/resource-usage-monitoring/)
