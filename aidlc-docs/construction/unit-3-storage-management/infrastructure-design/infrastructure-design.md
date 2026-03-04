# Infrastructure Design
## Unit 3: Storage Management Service

**Version**: 1.0  
**Date**: 2026-03-04

---

## 1. Infrastructure Overview

The Storage Management Service requires robust infrastructure to support high-throughput data writes, multiple database connections, and reliable message processing.

---

## 2. Compute Resources

### 2.1 Service Instances

**Development Environment**
- **Instances**: 1
- **CPU**: 2 cores
- **Memory**: 2GB
- **Purpose**: Local development and testing

**Staging Environment**
- **Instances**: 2
- **CPU**: 2 cores per instance
- **Memory**: 4GB per instance
- **Purpose**: Integration testing, UAT

**Production Environment**
- **Instances**: 3-10 (auto-scaling)
- **CPU**: 4 cores per instance
- **Memory**: 8GB per instance
- **Purpose**: Live system

### 2.2 Resource Allocation

```yaml
# Kubernetes Resource Requests and Limits
resources:
  requests:
    memory: "2Gi"
    cpu: "1000m"
  limits:
    memory: "4Gi"
    cpu: "2000m"
```

---

## 3. Database Infrastructure

### 3.1 PostgreSQL

**Configuration**
- **Version**: PostgreSQL 15+
- **Instance Type**: db.r6g.xlarge (AWS RDS) or equivalent
- **Storage**: 500GB SSD (gp3)
- **IOPS**: 12,000
- **Connections**: Max 200
- **Replication**: Multi-AZ with 1 read replica

**Connection String**
```
jdbc:postgresql://postgres-primary.example.com:5432/analytics?ssl=true&sslmode=require
```

**Backup Strategy**
- **Automated Backups**: Daily at 2 AM UTC
- **Retention**: 30 days
- **Point-in-Time Recovery**: Enabled
- **Backup Window**: 2:00 AM - 3:00 AM UTC

### 3.2 MongoDB

**Configuration**
- **Version**: MongoDB 6+
- **Deployment**: Replica Set (3 nodes)
- **Instance Type**: m6g.xlarge or equivalent
- **Storage**: 1TB SSD per node
- **Connections**: Max 200

**Connection String**
```
mongodb://mongo1.example.com:27017,mongo2.example.com:27017,mongo3.example.com:27017/analytics?replicaSet=rs0&ssl=true
```

**Backup Strategy**
- **Automated Snapshots**: Daily
- **Retention**: 30 days
- **Oplog**: 48 hours

### 3.3 TimescaleDB

**Configuration**
- **Version**: TimescaleDB 2.x on PostgreSQL 15
- **Instance Type**: db.r6g.large
- **Storage**: 200GB SSD
- **Connections**: Max 100

**Connection String**
```
jdbc:postgresql://timescaledb.example.com:5432/metrics?ssl=true
```

**Retention Policies**
- **Raw Metrics**: 90 days
- **Aggregated Metrics**: 1 year
- **Compression**: Enabled after 7 days

### 3.4 Snowflake

**Configuration**
- **Warehouse Size**: Medium (initially)
- **Auto-Suspend**: 5 minutes
- **Auto-Resume**: Enabled
- **Multi-Cluster**: Enabled (max 3 clusters)

**Connection String**
```
jdbc:snowflake://account.snowflakecomputing.com/?warehouse=ANALYTICS_WH&db=ANALYTICS&schema=PUBLIC
```

---

## 4. Message Broker Infrastructure

### 4.1 Apache Kafka

**Cluster Configuration**
- **Brokers**: 3 (production), 1 (dev/staging)
- **Instance Type**: kafka.m5.large or equivalent
- **Storage**: 500GB SSD per broker
- **Replication Factor**: 3 (production), 1 (dev/staging)
- **Min In-Sync Replicas**: 2 (production)

**Topics**
```yaml
topics:
  - name: data-ready-for-storage
    partitions: 12
    replication-factor: 3
    retention-ms: 604800000  # 7 days
    
  - name: storage-failed
    partitions: 6
    replication-factor: 3
    retention-ms: 2592000000  # 30 days
```

**Consumer Group**
```yaml
consumer-group:
  name: storage-management
  instances: 3
  max-poll-records: 500
  session-timeout-ms: 30000
```

---

## 5. Cache Infrastructure

### 5.1 Redis

**Configuration**
- **Version**: Redis 7+
- **Deployment**: Cluster mode (3 shards, 1 replica each)
- **Instance Type**: cache.r6g.large
- **Memory**: 13GB per node
- **Eviction Policy**: allkeys-lru

**Connection**
```yaml
spring:
  redis:
    cluster:
      nodes:
        - redis1.example.com:6379
        - redis2.example.com:6379
        - redis3.example.com:6379
    password: ${REDIS_PASSWORD}
    ssl: true
```

**Cache Usage**
- **Schema Cache**: TTL 10 minutes
- **Routing Rules Cache**: TTL 5 minutes
- **Database Health Status**: TTL 30 seconds

---

## 6. Networking

### 6.1 Network Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Public Subnet                           │
│  ┌──────────────────────────────────────────────────────┐   │
│  │           Load Balancer (ALB/NLB)                    │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────────┐
│                     Private Subnet (App)                     │
│  ┌──────────────────────────────────────────────────────┐   │
│  │   Storage Management Service Instances (3-10)        │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────────┐
│                   Private Subnet (Data)                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ PostgreSQL  │  │  MongoDB    │  │ TimescaleDB │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│  ┌─────────────┐  ┌─────────────┐                           │
│  │   Kafka     │  │   Redis     │                           │
│  └─────────────┘  └─────────────┘                           │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 Security Groups / Firewall Rules

**Service Instance Security Group**
- **Inbound**:
  - Port 8080 from Load Balancer
  - Port 8080 from monitoring (Prometheus)
- **Outbound**:
  - Port 5432 to PostgreSQL
  - Port 27017 to MongoDB
  - Port 9092 to Kafka
  - Port 6379 to Redis
  - Port 443 to Snowflake
  - Port 443 to external services

**Database Security Group**
- **Inbound**:
  - Port 5432 from Service Instance SG (PostgreSQL)
  - Port 27017 from Service Instance SG (MongoDB)
  - Port 6379 from Service Instance SG (Redis)
- **Outbound**: None required

---

## 7. Monitoring Infrastructure

### 7.1 Prometheus

**Configuration**
- **Instance Type**: t3.medium
- **Storage**: 100GB SSD
- **Retention**: 15 days
- **Scrape Interval**: 15 seconds

**Scrape Configuration**
```yaml
scrape_configs:
  - job_name: 'storage-management'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: keep
        regex: storage-management
    metrics_path: '/actuator/prometheus'
    scrape_interval: 15s
```

### 7.2 Grafana

**Configuration**
- **Instance Type**: t3.small
- **Storage**: 20GB SSD
- **Dashboards**: Pre-configured for service metrics

**Key Dashboards**
- Storage Management Overview
- Database Performance
- Kafka Consumer Lag
- Error Rates and Latency

### 7.3 ELK Stack

**Elasticsearch**
- **Nodes**: 3
- **Instance Type**: r6g.large
- **Storage**: 500GB SSD per node
- **Retention**: 30 days (hot), 90 days (warm)

**Logstash**
- **Instances**: 2
- **Instance Type**: t3.medium
- **Input**: Filebeat from service instances

**Kibana**
- **Instance**: 1
- **Instance Type**: t3.small

---

## 8. Deployment Infrastructure

### 8.1 Kubernetes Cluster

**Cluster Configuration**
- **Version**: Kubernetes 1.28+
- **Node Groups**:
  - **System**: 3 nodes (t3.medium) for system pods
  - **Application**: 3-10 nodes (t3.xlarge) for application pods
- **Networking**: Calico or AWS VPC CNI
- **Ingress**: NGINX Ingress Controller

**Namespaces**
```yaml
namespaces:
  - name: storage-management-dev
  - name: storage-management-staging
  - name: storage-management-prod
```

### 8.2 CI/CD Infrastructure

**Jenkins/GitLab CI**
- **Instance Type**: t3.large
- **Storage**: 100GB SSD
- **Agents**: 3 build agents

**Container Registry**
- **Service**: AWS ECR, Docker Hub, or Harbor
- **Repositories**:
  - storage-management-service
  - storage-management-service-dev

**Artifact Repository**
- **Service**: Nexus or Artifactory
- **Purpose**: Maven artifacts, Docker images

---

## 9. Disaster Recovery

### 9.1 Backup Infrastructure

**Backup Storage**
- **Service**: AWS S3, Azure Blob, or GCS
- **Bucket**: analytics-backups
- **Lifecycle Policy**:
  - Standard: 30 days
  - Glacier: 30-365 days
  - Delete: After 365 days

**Backup Schedule**
```yaml
backups:
  postgresql:
    frequency: daily
    time: "02:00 UTC"
    retention: 30 days
  
  mongodb:
    frequency: daily
    time: "03:00 UTC"
    retention: 30 days
  
  configuration:
    frequency: on-change
    retention: 90 days
```

### 9.2 Recovery Procedures

**RTO (Recovery Time Objective)**: 4 hours  
**RPO (Recovery Point Objective)**: 1 hour

**Recovery Steps**
1. Provision new infrastructure (if needed)
2. Restore databases from latest backup
3. Deploy service from last known good version
4. Verify data integrity
5. Resume operations

---

## 10. Cost Estimation

### 10.1 Monthly Cost Breakdown (Production)

| Component | Quantity | Unit Cost | Total |
|-----------|----------|-----------|-------|
| Service Instances (t3.xlarge) | 3-10 avg 5 | $150/month | $750 |
| PostgreSQL (db.r6g.xlarge) | 1 + 1 replica | $400/month | $800 |
| MongoDB (m6g.xlarge) | 3 nodes | $300/month | $900 |
| TimescaleDB (db.r6g.large) | 1 | $200/month | $200 |
| Snowflake | Medium warehouse | $400/month | $400 |
| Kafka (kafka.m5.large) | 3 brokers | $200/month | $600 |
| Redis (cache.r6g.large) | 6 nodes | $150/month | $900 |
| Load Balancer | 1 | $25/month | $25 |
| Monitoring (Prometheus, Grafana) | 2 instances | $100/month | $200 |
| ELK Stack | 5 instances | $500/month | $500 |
| Storage (SSD, backups) | Various | - | $500 |
| Data Transfer | - | - | $200 |
| **Total** | | | **$5,975/month** |

### 10.2 Cost Optimization

**Strategies**:
- Use reserved instances for stable workloads (30-40% savings)
- Auto-scaling for variable workloads
- Snowflake auto-suspend (save 70% on idle time)
- S3 lifecycle policies for backups
- Right-sizing instances based on actual usage

---

## 11. Infrastructure as Code

### 11.1 Terraform Configuration

**Directory Structure**
```
infrastructure/
├── modules/
│   ├── compute/
│   ├── database/
│   ├── networking/
│   └── monitoring/
├── environments/
│   ├── dev/
│   ├── staging/
│   └── prod/
└── main.tf
```

**Example: PostgreSQL Module**
```hcl
resource "aws_db_instance" "postgresql" {
  identifier           = "storage-management-postgres"
  engine               = "postgres"
  engine_version       = "15.3"
  instance_class       = var.instance_class
  allocated_storage    = 500
  storage_type         = "gp3"
  iops                 = 12000
  
  db_name              = "analytics"
  username             = var.db_username
  password             = var.db_password
  
  multi_az             = true
  publicly_accessible  = false
  
  backup_retention_period = 30
  backup_window          = "02:00-03:00"
  maintenance_window     = "sun:04:00-sun:05:00"
  
  vpc_security_group_ids = [aws_security_group.database.id]
  db_subnet_group_name   = aws_db_subnet_group.database.name
  
  tags = {
    Name        = "storage-management-postgres"
    Environment = var.environment
  }
}
```

---

## 12. Security Infrastructure

### 12.1 Secrets Management

**AWS Secrets Manager / HashiCorp Vault**
```yaml
secrets:
  - name: postgres-credentials
    keys:
      - username
      - password
  
  - name: mongodb-credentials
    keys:
      - username
      - password
  
  - name: kafka-credentials
    keys:
      - username
      - password
  
  - name: encryption-keys
    keys:
      - aes-key
```

### 12.2 Certificate Management

**TLS Certificates**
- **Service**: AWS Certificate Manager or Let's Encrypt
- **Domains**:
  - storage-api.example.com
  - storage-internal.example.com
- **Renewal**: Automatic

---

## Document Status

**Status**: Complete  
**Next**: Code Generation

---
