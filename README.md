# k8s-observability

> Production-ready Kubernetes observability stack with VictoriaMetrics, Grafana, and multi-cluster monitoring

[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.24+-326CE5?style=flat&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![VictoriaMetrics](https://img.shields.io/badge/VictoriaMetrics-1.116.0-orange?style=flat)](https://victoriametrics.com/)
[![Grafana](https://img.shields.io/badge/Grafana-11.2.0-F46800?style=flat&logo=grafana&logoColor=white)](https://grafana.com/)
[![GitHub](https://img.shields.io/badge/GitHub-jasaulakh1988-181717?style=flat&logo=github)](https://github.com/jasaulakh1988)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Jasvinder_Singh-0077B5?style=flat&logo=linkedin)](https://www.linkedin.com/in/jasvinder-singh-7b833b98/)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

## 🎯 Overview

A comprehensive, production-tested observability solution designed for multi-account Kubernetes environments. This repository provides everything needed to deploy a centralized metrics collection, storage, and visualization platform across multiple clusters.

### Key Features

- **🏗️ Centralized Architecture**: Single VictoriaMetrics cluster for all environments
- **🔄 Multi-Cluster Support**: Unified monitoring for dev, staging, and production
- **📊 High Availability**: Redundant components with auto-failover
- **🔐 Secure by Design**: RBAC, network policies, and encrypted storage
- **⚡ High Performance**: Optimized for large-scale metric ingestion
- **📈 Scalable**: Horizontally scalable components
- **🛠️ Production Ready**: Battle-tested configurations and best practices

## 🏛️ Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Infrastructure Cluster                       │
│                      (Centralized Monitoring Hub)                    │
│                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐ │
│  │  VictoriaMetrics │  │    Grafana HA    │  │    Exporters     │ │
│  │     Cluster      │  │  (Visualization) │  │  + vmagent       │ │
│  │                  │  │                  │  │ (Local Scraping) │ │
│  │  • vminsert (2)  │  │  • 2 Replicas    │  │                  │ │
│  │  • vmselect (2)  │  │  • PostgreSQL    │  │ • Node Exporter  │ │
│  │  • vmstorage (2) │  │  • EFS Storage   │  │ • Kube State     │ │
│  │  • vmauth (2)    │  │                  │  │                  │ │
│  │  • vmalert (2)   │  │                  │  │                  │ │
│  └────────▲─────────┘  └──────────────────┘  └──────────────────┘ │
│           │                                                          │
│           │ Remote Write (Authenticated)                            │
└───────────┼──────────────────────────────────────────────────────────┘
            │
            │
┌───────────┴──────────────────────────────────────────────────────────┐
│                                                                       │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │   Dev Cluster    │  │ Staging Cluster  │  │  Prod Cluster    │  │
│  │                  │  │                  │  │                  │  │
│  │  • Exporters     │  │  • Exporters     │  │  • Exporters     │  │
│  │  • vmagent       │  │  • vmagent       │  │  • vmagent       │  │
│  │    (Remote Write)│  │    (Remote Write)│  │    (Remote Write)│  │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘  │
│                                                                       │
│                    Remote Clusters (Application Workloads)           │
└───────────────────────────────────────────────────────────────────────┘
```

### Design Principles

- **Single Source of Truth**: One VictoriaMetrics cluster stores all metrics
- **Edge Collection**: vmagent in each cluster for local scraping
- **Centralized Visualization**: Shared Grafana for all environments
- **Labeled Data**: Cluster/environment labels for easy filtering
- **Secure Communication**: Authenticated remote write with TLS

## 📁 Repository Structure

```
k8s-observability/
├── 📄 README.md                          # You are here
│
├── 📂 docs/                              # Comprehensive documentation
│   ├── architecture.md                   # System architecture deep-dive
│   ├── victoria-metrics-setup.md         # VictoriaMetrics installation guide
│   ├── grafana-setup.md                  # Grafana HA deployment guide
│   ├── grafana-quick-reference.md        # Grafana quick commands
│   └── exporters-setup.md                # Exporters deployment guide
│
├── 📂 architecture/                      # Architecture diagrams
│   ├── diagrams/                         # Visual architecture
│   └── design-decisions.md               # Architecture rationale
│
├── 📂 infra-cluster/                     # Central monitoring cluster
│   ├── 📂 helm/                          # Helm chart deployments
│   │   ├── 📂 victoria-metrics/          # Metrics storage
│   │   │   ├── 📂 base/                  # Base configuration
│   │   │   │   ├── Chart.yaml            # Helm dependencies
│   │   │   │   ├── values.yaml           # Default values
│   │   │   │   └── 📂 infra/             # Environment overrides
│   │   │   │       └── values.yaml       # Production config
│   │   │   └── README.md                 # 📖 VictoriaMetrics guide
│   │   │
│   │   └── 📂 grafana/                   # Visualization platform
│   │       ├── 📂 base/                  # Base configuration
│   │       │   ├── Chart.yaml            # Helm dependencies
│   │       │   ├── values.yaml           # Default values
│   │       │   ├── grafana-efs-pvc.yaml  # Shared storage
│   │       │   └── 📂 infra/             # Environment overrides
│   │       │       └── values.yaml       # Production config
│   │       └── README.md                 # 📖 Grafana guide
│   │
│   ├── 📂 vmagent/                       # Local metrics collection
│   │   ├── vmagent.yaml                  # Deployment manifest
│   │   └── README.md                     # 📖 vmagent guide
│   │
│   └── 📂 exporters/                     # Metrics exporters
│       ├── node-exporter.yaml            # Host metrics
│       ├── kube-state-metrics.yaml       # K8s object metrics
│       └── README.md                     # 📖 Exporters guide
│
├── 📂 remote-clusters/                   # Dev/Staging/Prod clusters
│   ├── 📂 vmagent/                       # Remote write configuration
│   │   └── README.md                     # 📖 Coming soon
│   └── 📂 exporters/                     # Metrics exporters
│       └── README.md                     # 📖 Coming soon
│
├── 📂 examples/                          # Example configurations
│   ├── dev/                              # Development cluster
│   ├── staging/                          # Staging cluster
│   └── prod/                             # Production cluster
│
└── 📂 scripts/                           # Automation scripts
    ├── deploy-infra.sh                   # Deploy infra cluster
    ├── deploy-remote.sh                  # Deploy remote cluster
    └── validate-config.sh                # Configuration validation
```

## 🚀 Quick Start

### Prerequisites

- Kubernetes cluster (1.24+)
- `kubectl` and `helm` installed
- Cluster admin access
- AWS account (for EKS, EFS, Aurora)

### Infrastructure Cluster Setup

Deploy the central monitoring hub:

```bash
# 1. Clone repository
git clone https://github.com/jasaulakh1988/k8s-observability.git
cd k8s-observability

# 2. Create namespace
kubectl create namespace monitoring

# 3. Deploy VictoriaMetrics
cd infra-cluster/helm/victoria-metrics/base
helm dependency update
helm upgrade --install victoria-metrics . \
  --namespace monitoring \
  --values values.yaml \
  --values infra/values.yaml

# 4. Deploy Grafana
cd ../grafana/base
helm dependency update
helm upgrade --install grafana . \
  --namespace monitoring \
  --values values.yaml \
  --values infra/values.yaml

# 5. Deploy Exporters
cd ../../exporters
kubectl apply -f node-exporter.yaml
kubectl apply -f kube-state-metrics.yaml

# 6. Deploy vmagent
cd ../vmagent
kubectl apply -f vmagent.yaml
```

### Verification

```bash
# Check all components
kubectl get pods -n monitoring

# Access Grafana
kubectl port-forward svc/grafana 3000:80 -n monitoring
# Open: http://localhost:3000

# Access VictoriaMetrics
kubectl port-forward svc/victoria-metrics-vm-vmselect 8481:8481 -n monitoring
# Query: http://localhost:8481/select/0/prometheus/api/v1/query?query=up
```

## 📚 Documentation

### Getting Started

| Component | Quick Start | Detailed Guide |
|-----------|-------------|----------------|
| **VictoriaMetrics** | [README](infra-cluster/helm/victoria-metrics/README.md) | [Setup Guide](docs/victoria-metrics-setup.md) |
| **Grafana** | [README](infra-cluster/helm/grafana/README.md) | [Setup Guide](docs/grafana-setup.md) |
| **Exporters** | [README](infra-cluster/exporters/README.md) | [Setup Guide](docs/exporters-setup.md) |
| **vmagent** | [README](infra-cluster/vmagent/README.md) | - |

### Additional Resources

- **[Architecture Guide](docs/architecture.md)** - System design and data flow
- **[Grafana Quick Reference](docs/grafana-quick-reference.md)** - Common operations
- **[Troubleshooting](docs/troubleshooting.md)** - Common issues and solutions *(coming soon)*

## 🎯 Use Cases

### Centralized Multi-Cluster Monitoring
Monitor dev, staging, and production environments from a single Grafana instance with unified dashboards and alerting.

### High-Availability Observability
All critical components run with multiple replicas and automatic failover for 99.9%+ uptime.

### Cost-Effective at Scale
VictoriaMetrics provides 10x better compression than Prometheus, reducing storage costs significantly.

### Multi-Tenant Architecture
Isolate metrics by cluster/environment using labels and VMAuth for secure multi-team access.

## 🛠️ Technology Stack

| Component | Version | Purpose |
|-----------|---------|---------|
| **VictoriaMetrics** | 1.116.0 | Time-series database |
| **Grafana** | 11.2.0 | Visualization & dashboards |
| **vmagent** | 1.116.0 | Metrics collection |
| **Node Exporter** | 1.9.1 | Host metrics |
| **Kube State Metrics** | 2.16.0 | Kubernetes metrics |
| **PostgreSQL** | 15+ (Aurora) | Grafana backend |
| **EFS** | - | Shared storage |

## 📊 Monitoring Capabilities

### Infrastructure Metrics
- ✅ CPU, Memory, Disk, Network per node
- ✅ Kubernetes cluster health
- ✅ Pod/container resource usage
- ✅ Storage volume metrics

### Application Metrics
- ✅ Pod lifecycle and status
- ✅ Deployment rollout status
- ✅ Service endpoint health
- ✅ Custom application metrics

### Platform Metrics
- ✅ VictoriaMetrics performance
- ✅ Grafana usage statistics
- ✅ vmagent scrape health
- ✅ Karpenter node provisioning

## 🔒 Security Features

- **RBAC**: Role-based access control for all components
- **Network Policies**: Restricted pod-to-pod communication
- **Pod Security**: Non-root containers, read-only filesystems
- **Secrets Management**: External Secrets Operator with AWS Secrets Manager
- **TLS**: Encrypted communication for remote write
- **Authentication**: VMAuth for access control

## 📈 Performance & Scalability

### Metrics Capacity
- **Ingestion Rate**: 1M+ samples/second
- **Active Series**: 10M+ concurrent time series
- **Retention**: Configurable (default: 15 days)
- **Query Performance**: Sub-second for most queries

### Resource Requirements

**Minimum (Small Cluster - <10 nodes)**:
- VictoriaMetrics: 2 CPU, 4Gi memory
- Grafana: 500m CPU, 1Gi memory
- vmagent: 250m CPU, 512Mi memory

**Recommended (Medium Cluster - 10-50 nodes)**:
- VictoriaMetrics: 4 CPU, 8Gi memory
- Grafana: 2 CPU, 4Gi memory
- vmagent: 1 CPU, 2Gi memory

## 🤝 Contributing

Contributions are welcome! Please read our [Contributing Guide](CONTRIBUTING.md) for details on our code of conduct and the process for submitting pull requests.

### Git Commit Guidelines

We follow conventional commits for clear and structured commit history:

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types:**
- `feat`: New feature or enhancement
- `fix`: Bug fix
- `docs`: Documentation changes
- `config`: Configuration updates
- `refactor`: Code refactoring
- `chore`: Maintenance tasks

**Examples:**
```bash
feat(victoria-metrics): add HA configuration for vminsert
fix(grafana): resolve database connection timeout issue
docs(exporters): update node-exporter installation guide
config(vmagent): increase scrape interval to 60s
```

**Scope examples:** `victoria-metrics`, `grafana`, `vmagent`, `exporters`, `docs`, `scripts`

## 📝 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- [VictoriaMetrics](https://victoriametrics.com/) for the excellent time-series database
- [Grafana Labs](https://grafana.com/) for the visualization platform
- [Prometheus](https://prometheus.io/) for the exporters and metrics format
- The Kubernetes community for making this all possible

## 📞 Support

- **Documentation**: Check the [docs/](docs/) directory
- **Issues**: [GitHub Issues](https://github.com/jasaulakh1988/k8s-observability/issues)
- **Discussions**: [GitHub Discussions](https://github.com/jasaulakh1988/k8s-observability/discussions)
- **GitHub**: [@jasaulakh1988](https://github.com/jasaulakh1988)
- **LinkedIn**: [Jasvinder Singh](https://www.linkedin.com/in/jasvinder-singh-7b833b98/) - Platform Engineering

## 🗺️ Roadmap

- [x] VictoriaMetrics cluster deployment
- [x] Grafana HA with PostgreSQL
- [x] Node Exporter & Kube State Metrics
- [x] vmagent for local scraping
- [ ] Remote cluster setup (dev/staging/prod)
- [ ] Pre-configured dashboards
- [ ] Alert rules and notification channels
- [ ] Backup and disaster recovery procedures
- [ ] Terraform/CDK infrastructure as code
- [ ] CI/CD integration examples

---

<div align="center">

**[⭐ Star this repo](https://github.com/jasaulakh1988/k8s-observability)** if you find it useful!

Created and maintained by [Jasvinder Singh](https://www.linkedin.com/in/jasvinder-singh-7b833b98/)

[![GitHub followers](https://img.shields.io/github/followers/jasaulakh1988?style=social)](https://github.com/jasaulakh1988)
[![LinkedIn](https://img.shields.io/badge/Connect-LinkedIn-blue?style=social&logo=linkedin)](https://www.linkedin.com/in/jasvinder-singh-7b833b98/)

</div>
