# Splunk: Complete Guide

## Table of Contents
1. [Introduction to Splunk](#introduction)
2. [How Splunk Works](#how-splunk-works)
3. [Splunk Architecture & Components](#architecture)
4. [Core Concepts](#core-concepts)
5. [Docker Deployment](#docker-deployment)
6. [Distributed Splunk Architecture](#distributed-architecture)
7. [Use Cases & Real-World Examples](#use-cases)
8. [Configuration & Specifications](#configuration)
9. [AWS Integration](#aws-integration)
10. [Alternatives to Splunk](#alternatives)

---

## Introduction to Splunk {#introduction}

### What is Splunk?

Splunk is a powerful data platform designed to collect, index, analyze, and visualize machine-generated data from various sources in real-time. It transforms raw, unstructured log data into actionable insights, enabling organizations to monitor infrastructure, applications, and security systems.

### Why Splunk?

- **Real-time data processing**: Ingests and processes terabytes of data per day
- **Scalability**: Handles massive volumes of data across distributed environments
- **Flexibility**: Works with any data format (logs, JSON, XML, CSV)
- **Security**: Advanced threat detection and compliance monitoring
- **Operational Intelligence**: Dashboards, alerts, and reports for proactive monitoring
- **Enterprise-grade**: Used by enterprises like Coca-Cola, Intel, Vodafone, and NASA

### Splunk Editions

1. **Splunk Enterprise**: Self-hosted, on-premise deployment with full control
2. **Splunk Cloud**: SaaS solution with 100% uptime SLA, scales to 10TB/day
3. **Splunk Light**: Limited version for small deployments (no indexer clustering)
4. **Splunk Free**: Limited functionality, no forwarder support

---

## How Splunk Works {#how-splunk-works}

### The Three-Stage Data Pipeline

Splunk processes data through three main stages:

#### 1. **Data Collection Stage**
- Collects logs from multiple sources (files, APIs, network ports, scripts)
- Uses forwarders (lightweight agents) to push data from source systems
- Minimal processing at source to reduce overhead
- Can handle 1-2% CPU usage even at scale

#### 2. **Indexing Stage**
- Receives data from forwarders
- Parses raw data into structured events
- Applies transformations (timestamp identification, field extraction)
- Stores indexed data in optimized format for fast searching
- Creates buckets (directories) containing indexed data

#### 3. **Search & Analysis Stage**
- Users query indexed data via search interface
- Executes distributed searches across multiple indexers
- Merges results from multiple sources
- Generates reports, dashboards, and alerts
- Stores user-defined knowledge objects

### Data Flow Example

```
Raw Logs (Application, OS, Network) 
    ↓
Forwarder (Universal or Heavy)
    ↓ (TCP/UDP over network)
Indexer (Parsing + Indexing)
    ↓ (Stores in buckets)
Search Head (User Interface)
    ↓ (REST API)
User Dashboard/Reports/Alerts
```

---

## Splunk Architecture & Components {#architecture}

### Core Components

#### 1. **Forwarders** (Data Collection)

**Universal Forwarder (UF)**
- Lightweight agent (~10MB)
- Minimal CPU/memory footprint (1-2%)
- Only forwards raw data without processing
- **Best for**: High-scale environments, remote systems
- **Trade-off**: More bandwidth used, indexer handles all parsing

**Heavy Forwarder (HF)**
- Performs parsing and field extraction at source
- Intelligent data routing and cloning
- Larger footprint but reduces bandwidth
- **Best for**: Limited bandwidth scenarios, need pre-processing
- **Trade-off**: Uses more resources on source system

**Configuration Example (inputs.conf)**
```
[monitor:///var/log/apache2/access.log]
index = main
sourcetype = apache:access
source = /var/log/apache2/access.log
```

#### 2. **Indexers** (Data Storage & Indexing)

- Transform incoming data into searchable events
- Create time-based indexes organized in buckets
- Generate TSIDX (time series index) files for fast searching
- Support data replication for redundancy
- Can form clusters for high availability

**Default Receiving Port**: 9997 (TCP)
**Default Web UI Port**: 8000

**Index Storage Structure**
```
$SPLUNK_HOME/var/lib/splunk/main/db/
    ├── bucket_0/ (Hot - currently writing)
    │   ├── rawsize (compressed raw data)
    │   ├── journal.gz
    │   └── metadata
    ├── bucket_1/ (Warm - no longer receiving)
    │   └── rawsize.tsidx (indexed data)
    └── bucket_N/ (Cold - not used recently)
```

**Bucket States**
- **Hot**: Currently receiving data
- **Warm**: Closed and no longer receiving data
- **Cold**: Moved to cold storage (archival)
- **Frozen**: Deleted (or sent to archive)

#### 3. **Search Heads** (User Interface & Search)

- Web-based UI for searching and analyzing data
- Distributes searches to indexers (search peers)
- Merges results from multiple indexers
- Stores dashboards, reports, alerts
- Can be clustered for high availability

**Key Features**
- Advanced search language (SPL - Splunk Processing Language)
- Real-time and scheduled searches
- Alert triggering based on conditions
- Knowledge objects (reports, dashboards, lookups)

**Default Web Port**: 8000

#### 4. **Deployment Server**

- Centralized configuration management
- Distributes configurations to deployment clients
- Manages apps, updates, and policies
- Reduces manual configuration across fleet

**Use Case**: Managing thousands of forwarders/indexers

#### 5. **License Master**

- Manages Splunk licensing
- Monitors data ingestion (GB/day limits)
- Tracks license compliance
- Sends license warnings

---

## Core Concepts {#core-concepts}

### Indexes

**Definition**: A repository of time-sorted, searchable data

**Common Indexes**
- `main`: Default index for most data
- `_internal`: Splunk's own operational logs
- `_audit`: User actions and configuration changes
- `_syslog`: System logs

**Index Configuration (indexes.conf)**
```
[myindex]
homePath = $SPLUNK_HOME/var/lib/splunk/myindex/db
coldPath = $SPLUNK_HOME/var/lib/splunk/myindex/colddb
datatype = event
maxKBps = 40000
maxConcurrentOptimizes = 6
```

### Events

**Definition**: A single log entry or data point

**Event Components**
- **timestamp**: When the event occurred
- **source**: Where the event came from (/var/log/app.log)
- **sourcetype**: Type of data (apache:access, syslog, json)
- **host**: System that generated the event
- **fields**: Extracted key-value pairs

**Example Event**
```
timestamp=2024-03-19 15:23:45.123
host=web-server-01
source=/var/log/apache/access.log
sourcetype=apache:access
method=GET
status=200
url=/api/users
response_time=145ms
```

### Fields

**Automatic Fields** (extracted automatically)
- `host`: System generating the event
- `source`: File path or data source
- `sourcetype`: Type of log
- `_time`: Event timestamp
- `_raw`: Original unprocessed event

**Extracted Fields** (custom extraction)
```
[source::/var/log/app.log]
EXTRACT-method = method=(?<http_method>\w+)
EXTRACT-status = status=(?<http_status>\d{3})
EXTRACT-response_time = response_time=(?<rt>\d+)ms
```

### Sourcetypes

Configuration that tells Splunk how to parse data

**Common Sourcetypes**
```
apache:access
apache:error
syslog
json
csv
windows:security
```

---

## Docker Deployment {#docker-deployment}

### Prerequisites
- Docker installed and running
- Minimum 8GB RAM, 4 CPU cores
- Network connectivity between containers

### Quick Start: Single Container

**Basic Single Instance**
```bash
docker pull splunk/splunk:latest

docker run -d \
  --name splunk \
  -p 8000:8000 \
  -p 9997:9997 \
  -e SPLUNK_START_ARGS='--accept-license' \
  -e SPLUNK_PASSWORD='MySecurePassword123!' \
  -e SPLUNK_GENERAL_TERMS='--accept-sgt-current-at-splunk-com' \
  splunk/splunk:latest

# Access Splunk Web
# URL: http://localhost:8000
# Username: admin
# Password: MySecurePassword123!
```

**Port Mappings**
- `8000`: Splunk Web UI
- `9997`: Forwarder receiving port
- `8089`: Splunk management port

### Docker Compose for Distributed Environment

**docker-compose.yml**
```yaml
version: '3.8'

services:
  # Indexer 1
  indexer1:
    image: splunk/splunk:latest
    container_name: splunk-indexer1
    environment:
      SPLUNK_START_ARGS: '--accept-license'
      SPLUNK_PASSWORD: 'SecurePass123!'
      SPLUNK_GENERAL_TERMS: '--accept-sgt-current-at-splunk-com'
      SPLUNK_ROLE: 'splunk_indexer'
      SPLUNK_CLUSTER_MASTER_URL: 'https://cluster-master:8089'
      SPLUNK_CLUSTER_LABEL: 'prod_cluster'
      SPLUNK_INDEXER_DISCOVERY: 'cluster-master:8089'
    ports:
      - "9997:9997"
      - "8001:8000"
    volumes:
      - indexer1-data:/opt/splunk/var
      - ./defaults/default.yml:/tmp/defaults/default.yml
    networks:
      - splunk-net
    healthcheck:
      test: ['CMD', 'curl', '-f', 'http://localhost:8000/api/server/info']
      interval: 30s
      timeout: 10s
      retries: 3

  # Indexer 2
  indexer2:
    image: splunk/splunk:latest
    container_name: splunk-indexer2
    environment:
      SPLUNK_START_ARGS: '--accept-license'
      SPLUNK_PASSWORD: 'SecurePass123!'
      SPLUNK_GENERAL_TERMS: '--accept-sgt-current-at-splunk-com'
      SPLUNK_ROLE: 'splunk_indexer'
      SPLUNK_CLUSTER_MASTER_URL: 'https://cluster-master:8089'
      SPLUNK_CLUSTER_LABEL: 'prod_cluster'
      SPLUNK_INDEXER_DISCOVERY: 'cluster-master:8089'
    ports:
      - "9998:9997"
      - "8002:8000"
    volumes:
      - indexer2-data:/opt/splunk/var
      - ./defaults/default.yml:/tmp/defaults/default.yml
    networks:
      - splunk-net
    depends_on:
      - cluster-master

  # Cluster Master (Indexer Cluster Manager)
  cluster-master:
    image: splunk/splunk:latest
    container_name: splunk-cluster-master
    environment:
      SPLUNK_START_ARGS: '--accept-license'
      SPLUNK_PASSWORD: 'SecurePass123!'
      SPLUNK_GENERAL_TERMS: '--accept-sgt-current-at-splunk-com'
      SPLUNK_ROLE: 'splunk_cluster_master'
      SPLUNK_CLUSTER_LABEL: 'prod_cluster'
      SPLUNK_REPLICATION_FACTOR: 2
      SPLUNK_SEARCH_FACTOR: 2
    ports:
      - "9999:9997"
      - "8003:8000"
    volumes:
      - cluster-master-data:/opt/splunk/var
      - ./defaults/default.yml:/tmp/defaults/default.yml
    networks:
      - splunk-net

  # Search Head
  search-head:
    image: splunk/splunk:latest
    container_name: splunk-search-head
    environment:
      SPLUNK_START_ARGS: '--accept-license'
      SPLUNK_PASSWORD: 'SecurePass123!'
      SPLUNK_GENERAL_TERMS: '--accept-sgt-current-at-splunk-com'
      SPLUNK_ROLE: 'splunk_search_head'
      SPLUNK_INDEXER_DISCOVERY: 'cluster-master:8089'
      SPLUNK_SEARCH_VOLUME_DEFAULT: 'primary'
    ports:
      - "8000:8000"
    volumes:
      - search-head-data:/opt/splunk/var
      - ./defaults/default.yml:/tmp/defaults/default.yml
    networks:
      - splunk-net
    depends_on:
      - indexer1
      - indexer2
      - cluster-master

  # Universal Forwarder
  forwarder:
    image: splunk/universalforwarder:latest
    container_name: splunk-forwarder
    environment:
      SPLUNK_START_ARGS: '--accept-license'
      SPLUNK_PASSWORD: 'SecurePass123!'
      SPLUNK_GENERAL_TERMS: '--accept-sgt-current-at-splunk-com'
      SPLUNK_FORWARD_SERVER: 'indexer1:9997'
      SPLUNK_ADD: unix
    volumes:
      - /var/log:/host/var/log:ro
      - ./forwarder-config:/tmp/defaults/default.yml
    networks:
      - splunk-net
    depends_on:
      - indexer1

volumes:
  indexer1-data:
  indexer2-data:
  cluster-master-data:
  search-head-data:

networks:
  splunk-net:
    driver: bridge
```

**default.yml Configuration**
```yaml
splunk:
  # License
  license_master_url: https://license-master:8089

  # Indexing
  indexes:
    - name: main
      datatype: event
      maxKBps: 40000

  # Receiving (for forwarders)
  inputs:
    tcp:
      default:
        port: 9997

  # Outputs (for forwarders)
  outputs:
    tcpout:
      default:
        servers:
          - indexer1:9997
          - indexer2:9997
        loadBalanceType: round-robin

  # SSL Configuration
  ssl:
    ca: /opt/splunk/etc/apps/deployment-client/metadata/ca.crt
    server_cert: /opt/splunk/etc/apps/deployment-client/metadata/server.crt
    server_key: /opt/splunk/etc/apps/deployment-client/metadata/server.key
    sslConfig:
      requireClientCert: false
```

### Docker Volume Management

**Named Volumes vs Bind Mounts**
```bash
# Named volumes (recommended for production)
docker volume create splunk-data

docker run -d \
  -v splunk-data:/opt/splunk/var \
  splunk/splunk:latest

# Bind mounts (for development)
docker run -d \
  -v /local/path:/opt/splunk/var \
  splunk/splunk:latest
```

### Multi-Host Docker Setup with Docker Swarm

```bash
# Initialize swarm on primary node
docker swarm init

# Deploy Splunk stack
docker stack deploy -c docker-compose.yml splunk

# Check services
docker service ls

# Scale indexers
docker service scale splunk_indexer=5

# Monitor logs
docker service logs splunk_search-head
```

---

## Distributed Splunk Architecture {#distributed-architecture}

### Topology 1: Simple Distributed Setup

```
Multiple Forwarders
    ↓ (TCP 9997)
    ├→ Indexer 1
    ├→ Indexer 2
    └→ Indexer 3
    ↓ (Distributed Search)
Search Head (Admin Access)
```

**Use Case**: Small to medium deployments, multiple data sources, centralized search

### Topology 2: Indexer Cluster (HA)

```
Forwarders (Load Balanced)
    ↓
Indexer Cluster (3+ Indexers)
    ├→ Indexer 1 (Peer)
    ├→ Indexer 2 (Peer)
    ├→ Indexer 3 (Peer)
    └→ Cluster Master (Manager Node)
    ↓
Search Head Cluster
    ├→ Search Head 1
    ├→ Search Head 2
    └→ Cluster Manager
```

**Benefits**
- High availability (peer replication)
- Data redundancy (multiple copies)
- Automatic failover
- Distributed search

**Configuration: Cluster Master (server.conf)**
```
[clustering]
mode = master
cluster_label = prod_cluster
pass4SymmKey = your-cluster-secret-key
replication_factor = 3
search_factor = 2
heartbeat_timeout = 60
cxn_timeout = 30
rep_cxn_timeout = 30
```

**Configuration: Indexer Peers (server.conf)**
```
[clustering]
mode = peer
master_uri = https://cluster-master:8089
pass4SymmKey = your-cluster-secret-key
cxn_timeout = 30
```

### Topology 3: Search Head Cluster

```
Multiple Search Heads
    ├→ Search Head 1 (Peer)
    ├→ Search Head 2 (Peer)
    ├→ Search Head 3 (Peer)
    └→ Search Head Cluster Manager
    ↓
Shared Knowledge (Dashboards, Reports)
```

**Configuration: Cluster Manager (server.conf)**
```
[shclustering]
conf_deploy_fetch_url = https://deployment-server:8089
disabled = false
election_timeout_ms = 60000
heartbeat_timeout = 60
pass4SymmKey = your-shc-secret-key
```

**Configuration: Search Head Peers (server.conf)**
```
[shclustering]
conf_deploy_fetch_url = https://deployment-server:8089
disabled = false
pass4SymmKey = your-shc-secret-key
shcluster_label = sh_cluster_1
server_uri = https://search-head-1:8089
```

### Topology 4: Multi-Site Indexer Cluster (Geo-Distributed)

```
Site 1 (US-East)          Site 2 (EU-West)
├→ Indexer 1              ├→ Indexer 4
├→ Indexer 2              ├→ Indexer 5
└→ Indexer 3              └→ Indexer 6
  ↓ (Cluster Master)        ↓
  └─ Replication across sites
```

**Configuration: Multi-Site (server.conf)**
```
[clustering]
mode = master
cluster_label = prod_cluster
site_replication_factor = origin:2, total:3
site_search_factor = origin:1, total:2
pass4SymmKey = your-secret-key

[clustering]
mode = peer
site = site1
master_uri = https://cluster-master:8089
```

---

## Use Cases & Real-World Examples {#use-cases}

### Use Case 1: E-Commerce Application Monitoring

**Scenario**: Monitor an online store with 10 million daily active users

**Components**
- 100+ web servers (Universal Forwarders)
- 50+ application servers (Heavy Forwarders)
- 3 Indexers (clustered)
- 2 Search Heads (for redundancy)
- Deployment Server

**Data Collection**
```ini
# On each web server
[monitor:///var/log/nginx/access.log]
sourcetype = nginx:access
index = web_metrics

[monitor:///var/log/nginx/error.log]
sourcetype = nginx:error
index = web_errors

# On app servers (Heavy Forwarder)
[monitor:///var/log/app/transaction.log]
sourcetype = app:transaction
TRANSFORMS = filter_debug_logs

[filter_debug_logs]
REGEX = DEBUG
DEST_KEY = queue
FORMAT = nullQueue
```

**Key Metrics Tracked**
- Page load times (latency)
- HTTP status codes (errors)
- Database response times
- Payment processing errors
- User session tracking

**Search Query Example**
```spl
index=web_metrics sourcetype=nginx:access status=500
| stats count by host
| sort - count
```

**Dashboards**
- Real-time sales transactions
- Performance anomalies
- Error rate alerts
- Capacity planning

**Alerts**
```spl
index=web_metrics sourcetype=nginx:access status>=500
| stats count
| where count > 50
```

### Use Case 2: Security Information & Event Management (SIEM)

**Scenario**: Monitor security events across 1000+ servers

**Data Sources**
- Firewall logs (Palo Alto, Cisco ASA)
- Intrusion detection (Suricata, Snort)
- Windows Event Logs (AD, Security)
- Web Application Firewall logs
- VPN connection logs
- SSH/failed login attempts

**Architecture**
```
Heavy Forwarders (1 per site)
    ↓
Indexers (Dedicated for security)
    ↓
Search Head (Security team)
```

**Index Structure**
```
index=security sourcetype=windows:security EventID=4625
(failed login attempts)

index=security sourcetype=firewall action=dropped
(blocked connections)

index=security sourcetype=ids alert=true severity>=3
(intrusion alerts)
```

**Threat Detection Search**
```spl
index=security sourcetype=windows:security EventID=4625
| stats count by user, host
| where count > 10
| alert_on violation
```

### Use Case 3: Infrastructure Monitoring (DevOps)

**Scenario**: Monitor Kubernetes cluster with 500+ containers

**Data Collection**
- Pod logs (stdout/stderr)
- Node metrics (CPU, memory, disk)
- Container orchestration events
- Service mesh logs (Istio)
- Database performance metrics

**Forwarder Configuration**
```yaml
# On Kubernetes nodes (DaemonSet)
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: splunk-forwarder
spec:
  template:
    spec:
      containers:
      - name: splunk-uf
        image: splunk/universalforwarder:latest
        volumeMounts:
        - name: pod-logs
          mountPath: /var/log/pods
          readOnly: true
        - name: splunk-config
          mountPath: /opt/splunk/etc/apps/Splunk_TA_kubernetes/
```

**Key Metrics**
```spl
# Pod restart tracking
index=kubernetes sourcetype=pod_event action=restart
| stats count by pod_name, namespace
| where count > 5

# Node capacity
index=kubernetes sourcetype=node_metrics
| stats avg(cpu_percent) as avg_cpu, max(memory_percent) as max_mem by node
| where avg_cpu > 80 or max_mem > 85

# Deploy tracking
index=kubernetes sourcetype=deployment event_type=update
| stats count by deployment, namespace, version
```

### Use Case 4: Compliance & Audit Logging

**Scenario**: Financial institution requiring PCI-DSS, SOX, GDPR compliance

**Requirements**
- Immutable audit trail
- 7+ years data retention
- Failed access attempts tracking
- Data access monitoring
- Configuration change tracking

**Implementation**
```ini
# Audit index (immutable)
[audit_index]
datatype = event
maxKBps = unlimited
coldPath = /archive/audit/colddb
frozenTimePeriodInSeconds = 2147483647
maxConcurrentOptimizes = 0

# Search filtering
[source::/var/log/auth.log]
sourcetype = linux:auth
index = audit_index
TRANSFORMS = forward_to_central_logging
```

**Search Queries**
```spl
# Failed login attempts (potential breach)
index=audit_index sourcetype=linux:auth result=failure
| stats count by user, source_ip
| where count > 5
| alert_on breach_attempt

# Configuration changes (compliance)
index=audit_index action=modified object_type=system_config
| stats values(modified_by) as who, count by object
```

---

## Configuration & Specifications {#configuration}

### Core Configuration Files

#### 1. inputs.conf (Data Collection)

```ini
# Monitor a file
[monitor:///var/log/app.log]
index = main
sourcetype = app:log
source = /var/log/app.log

# Monitor a directory
[monitor:///var/log/apache2/]
index = main
sourcetype = apache:access
recursive = true

# Listen on TCP port
[tcpin://:9998]
connection_host = ip
sourcetype = syslog

# Listen on UDP port
[udpin://:514]
sourcetype = syslog

# Script input
[script:///usr/local/bin/get_data.sh]
interval = 60
sourcetype = custom:script
index = main
```

#### 2. outputs.conf (Data Forwarding)

```ini
# Forward to indexers
[tcpout]
defaultGroup = indexers
forwardedindex.filter.disable = true

[tcpout:indexers]
server = indexer1:9997, indexer2:9997, indexer3:9997
maxConnectionsPerIndexer = 2
maxQueueSize = 512KB
autoLBFrequency = 30
autoLBVolume = 10485760

# Load balancing with persistence
[tcpout:indexers]
loadBalanceSetting = persistentVolumeBased
volumeOnlyOnLoad = 5
forwardedindex.0.whitelist.load = _.*
forwardedindex.1.whitelist.load = (main|_internal|history|summary)
```

#### 3. props.conf (Parsing Rules)

```ini
[apache:access]
TRANSFORMS = parse_apache_access
FIELDALIAS-url = request as url
FIELDALIAS-code = status as http_code

[json_logs]
TRANSFORMS-routing = route_by_severity
AUTO_KV_JSON = true
KV_MODE = json
TIMESTAMP_FIELDS = @timestamp

# Extraction
[custom:app]
EXTRACT-method = method=(?<method>\w+)
EXTRACT-response = response=(?<response_time>\d+)ms
```

#### 4. transforms.conf (Data Transformation)

```ini
# Parsing transformation
[parse_apache_access]
REGEX = (?P<host>[\w\.]+)\s+-\s+\[(?P<timestamp>[^\]]*)\]\s+"(?P<method>\w+)\s+(?P<uri>[^\s]*)\s+(?P<protocol>[^"]*)"\s+(?P<status>\d+)\s+(?P<bytes>\d+)

# Routing transformation
[route_by_severity]
REGEX = severity=(ERROR|WARN|INFO|DEBUG)
DEST_KEY = _MetaData:Index
FORMAT = $1_logs

# Filtering
[nullqueue_filter]
REGEX = DEBUG|TEST
DEST_KEY = queue
FORMAT = nullQueue
```

#### 5. limits.conf (Performance Tuning)

```ini
[search]
max_mem_mb = 400
max_rawsize_perchunk = 100000000
max_chunk_queue_size = 20000000

[thruput]
maxKBps = 100000  # 100 MB/s

[indexer]
maxKBps = 40000   # Per indexer limit
```

#### 6. server.conf (Splunk Settings)

```ini
# General
[general]
pass4SymmKey = $7$ENCRYPTED_VALUE
server_ui = Lite

# License
[license]
master_uri_ = https://license-master:8089
connection_timeout = 20

# SSL/TLS
[sslConfig]
enableSSL = true
sslVersions = tls1.2, tls1.3
serverCert = /opt/splunk/etc/apps/ssl/server.pem
requireClientCert = false

# Clustering
[clustering]
mode = peer
master_uri = https://cluster-master:8089
pass4SymmKey = $7$ENCRYPTED_CLUSTER_KEY
replication_port = 8080
```

### Performance Tuning Specifications

#### Memory Configuration
```
# Search head (8 concurrent searches)
splunk_home/etc/system/local/limits.conf:

[search]
max_mem_mb = 400

[scheduler]
max_searches_per_cpu = 4
```

#### Storage Specifications
```
Index size = (Raw Data Size) × (Compression Ratio)
Compression Ratio ≈ 10:1 to 100:1 depending on data type

Example:
1 TB raw logs → 10-100 GB indexed data
```

#### Network Optimization
```ini
[tcpout:indexers]
maxConnectionsPerIndexer = 2
maxQueueSize = 512KB
tcpSendBufSz = 4096KB
sslVerifyServerCert = false  # Disable for internal traffic
```

---

## AWS Integration {#aws-integration}

### Splunk Cloud on AWS

Splunk Cloud is backed by a 100% uptime SLA and scales to over 10TB/day in a highly secure environment.

#### Architecture
```
AWS VPC
├── EC2 Instances (Applications)
│   └── CloudWatch Agent / Splunk Forwarder
├── ALB/NLB
│   └── Splunk Forwarder
├── RDS Database
│   └── Splunk DB Connect
├── Lambda
│   └── CloudWatch Logs → Kinesis Firehose
└── VPC Flow Logs → S3 → Splunk
```

#### Setup Steps

**1. Create Splunk Cloud Instance**
```
AWS Console → Splunk Cloud Setup
- Create workspace
- Configure authentication
- Enable auto-scaling
- Set data retention (7-90 days)
```

**2. Install Splunk Universal Forwarder on EC2**
```bash
# On EC2 instance
wget -O splunk-uf.tgz \
  'https://www.splunk.com/bin/splunk/DownloadActivityServlet?architecture=x86_64&platform=Linux&version=9.0.0&wget=true'

tar xzf splunk-uf.tgz -C /opt

# Configure forwarding
cat > /opt/splunk/etc/system/local/outputs.conf <<EOF
[tcpout]
defaultGroup = splunk_cloud

[tcpout:splunk_cloud]
server = your-instance.splunkcloud.com:9997
clientCert = /opt/splunk/etc/apps/deployment-client/metadata/client.pem
sslVerifyServerCert = true
EOF

# Start forwarder
/opt/splunk/bin/splunk start --accept-license --answer-yes --no-prompt
```

**3. Configure AWS Data Sources**

**CloudWatch Integration**
```python
import boto3
import json

# Fetch CloudWatch logs
logs_client = boto3.client('logs')

response = logs_client.describe_log_groups()

for log_group in response['logGroups']:
    # Send to Splunk HEC
    print(f"Log Group: {log_group['logGroupName']}")
    
    # Use Splunk HEC endpoint
    # https://your-instance.splunkcloud.com:8088/services/collector
```

**VPC Flow Logs → Splunk**
```
1. Enable VPC Flow Logs
2. Publish to CloudWatch Logs
3. Subscribe to Splunk endpoint
   Subscription filter: [version, account, interface_id, srcaddr, dstaddr, srcport, dstport, protocol, packets, bytes, windowstart, windowend, action, log_status]
4. Configure Splunk index for network:flow
```

**4. Set Up AWS Data Add-on**
```bash
# Install Splunk Add-on for AWS
# https://splunkbase.splunk.com/app/1876/

# Configuration: aws.conf
[aws-cloudwatch]
account = default
region = us-east-1
log_groups = /aws/lambda/my-function
interval = 300
sourcetype = aws:cloudwatch

[aws-s3]
sqs_queue_url = https://sqs.amazonaws.com/123456789/my-queue
aws_account = default
sourcetype = aws:s3
```

### Cost Optimization on AWS

**Data Filtering Strategy**
```spl
# Reduce data ingestion with filtering
[transform]
REGEX = DEBUG|TEST  # Exclude DEBUG/TEST logs
DEST_KEY = queue
FORMAT = nullQueue
```

**Archival Strategy**
```
Hot Index (Recent 1 day)
  ↓
Warm Index (1-30 days)
  ↓
Cold Index (31-365 days on S3)
  ↓
Frozen/Delete (>365 days)

Cost Reduction: Hot → Cold saves 90% storage cost
```

**AWS Budget Alert**
```bash
# Monitor Splunk Cloud data ingestion
aws cloudwatch get-metric-statistics \
  --namespace SplunkCloud \
  --metric-name DataIngestionGB \
  --start-time 2024-03-01T00:00:00Z \
  --end-time 2024-03-31T23:59:59Z \
  --period 86400 \
  --statistics Sum
```

---

## Alternatives to Splunk {#alternatives}

### Comparison Matrix

| Feature | Splunk | Datadog | ELK | New Relic | Graylog |
|---------|--------|---------|-----|-----------|---------|
| **Log Management** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Cloud Native** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **APM** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| **Scalability** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Ease of Use** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **SIEM Capability** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| **Pricing** | 💰💰💰 | 💰💰💰 | 💰 | 💰💰 | 💰 |
| **Open Source** | ❌ | ❌ | ✅ | ❌ | ✅ |

### Top Alternatives

#### 1. **Datadog**

Datadog is a more modern, cloud-native monitoring and observability tool designed to offer full-stack observability across infrastructure, applications, and services, known for its simplicity, real-time monitoring, and tight integrations with cloud platforms like AWS, Google Cloud, and Azure.

**Strengths**
- ✅ Cloud-native with AWS/Azure/GCP integrations
- ✅ Superior APM capabilities
- ✅ Quick deployment (15 minutes vs weeks for Splunk)
- ✅ Intuitive UI (better for beginners)
- ✅ 700+ out-of-box integrations

**Weaknesses**
- ❌ Expensive at scale (per-host pricing)
- ❌ Less customization than Splunk
- ❌ Cloud-only (no on-premise option)
- ❌ Weak SIEM capabilities

**Best For**: Cloud-native organizations, DevOps teams, real-time APM

**Example Setup**
```bash
# Install Datadog Agent
DD_AGENT_MAJOR_VERSION=7 DD_API_KEY=YOUR_API_KEY DD_SITE="datadoghq.com" \
bash -c "$(curl -L https://s3.amazonaws.com/dd-agent/scripts/install_agent.sh)"

# Monitor logs
tags: env:prod,service:api,version:1.0
```

#### 2. **Elasticsearch/ELK Stack**

**What is ELK?**
- **E**lasticsearch: Search and indexing engine
- **L**ogstash: Data processing pipeline
- **K**ibana: Visualization layer

**Strengths**
- ✅ Open-source and free
- ✅ Highly customizable
- ✅ Excellent for large-scale deployments
- ✅ Strong community support
- ✅ Can be self-hosted

**Weaknesses**
- ❌ Steep learning curve
- ❌ Requires significant operational overhead
- ❌ SIEM features require additional tools
- ❌ Complex to manage and tune

**Best For**: Tech teams with DevOps expertise, cost-conscious organizations

**Docker Setup**
```yaml
version: '3.8'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.0.0
    environment:
      - discovery.type=single-node
    ports:
      - "9200:9200"
    volumes:
      - es-data:/usr/share/elasticsearch/data

  logstash:
    image: docker.elastic.co/logstash/logstash:8.0.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    ports:
      - "5000:5000"

  kibana:
    image: docker.elastic.co/kibana/kibana:8.0.0
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_HOSTS: "http://elasticsearch:9200"

volumes:
  es-data:
```

#### 3. **New Relic**

**Strengths**
- ✅ Superior APM and RUM capabilities
- ✅ Excellent AI-driven anomaly detection
- ✅ Intuitive interface
- ✅ Good for microservices
- ✅ Free tier available

**Weaknesses**
- ❌ Log management not as strong as Splunk
- ❌ Expensive at enterprise scale
- ❌ SIEM features limited

**Best For**: Development teams, microservices architectures, APM-focused monitoring

#### 4. **Grafana Loki**

**What is Loki?**
- Lightweight log aggregation system
- Label-based indexing (like Prometheus)
- Part of Grafana ecosystem

**Strengths**
- ✅ Low resource consumption
- ✅ Works well with Kubernetes
- ✅ Integration with Prometheus
- ✅ Cost-effective
- ✅ Open source

**Weaknesses**
- ❌ Limited query language (LogQL)
- ❌ Not suited for complex log analysis
- ❌ Smaller community than ELK
- ❌ No SIEM features

**Best For**: Kubernetes environments, label-based monitoring

#### 5. **Amazon CloudWatch**

Amazon CloudWatch is ideal for monitoring AWS resources and helps gain system-wide visibility into resource utilization, application performance, and operational health.

**Strengths**
- ✅ Native AWS integration
- ✅ Seamless with EC2, Lambda, RDS, etc.
- ✅ Low cost for AWS workloads
- ✅ Simple to set up

**Weaknesses**
- ❌ AWS-only ecosystem
- ❌ Limited cross-platform support
- ❌ Less advanced analytics
- ❌ Expensive outside AWS

**Best For**: AWS-only organizations, simple logging needs

#### 6. **SigNoz (Open Source)**

**Strengths**
- ✅ Open source with transparent pricing
- ✅ Full-stack observability (metrics, logs, traces)
- ✅ OpenTelemetry native
- ✅ Cost-effective ($0.1/million samples)

**Weaknesses**
- ❌ Smaller community
- ❌ Fewer integrations than Splunk
- ❌ Still evolving

**Best For**: Cost-conscious startups, companies wanting open-source

### Decision Matrix for Choosing Alternative

```
Q1: Budget constraint?
├─ YES → ELK / Grafana Loki / SigNoz
└─ NO → Datadog / Splunk / New Relic

Q2: Cloud-native architecture?
├─ YES → Datadog / New Relic / Grafana Loki
└─ NO → Splunk / ELK

Q3: Heavy SIEM/Security requirements?
├─ YES → Splunk / Logz.io
└─ NO → Datadog / New Relic

Q4: AWS-only environment?
├─ YES → CloudWatch (+ Splunk/Datadog for advanced)
└─ NO → Multi-cloud option (Datadog / Splunk)

Q5: Team DevOps expertise?
├─ HIGH → ELK / SigNoz
└─ LOW → Datadog / New Relic / Splunk Cloud
```

---

## Summary & Best Practices

### When to Use Splunk
1. ✅ Large-scale log management (TB+ per day)
2. ✅ SIEM and security monitoring
3. ✅ Complex data analysis and correlation
4. ✅ Compliance and audit requirements
5. ✅ Hybrid/on-premise environments

### When to Use Alternatives
1. ✅ Cloud-only, modern infrastructure → Datadog
2. ✅ Cost-sensitive → ELK / SigNoz
3. ✅ Kubernetes-focused → Grafana Loki
4. ✅ AWS-only → CloudWatch
5. ✅ APM-heavy → New Relic

### Production Best Practices

**Architecture**
- Deploy indexer clusters (minimum 3 nodes)
- Use search head clusters for high availability
- Implement load balancing for forwarders
- Separate roles (forwarder, indexer, search head)

**Security**
- Enable SSL/TLS for all communications
- Use strong authentication (LDAP/AD)
- Implement role-based access control (RBAC)
- Encrypt sensitive data at rest and in transit

**Performance**
- Monitor indexing rate (max_KBps)
- Tune search performance with accelerated data models
- Archive old data to cold storage
- Use deployment server for configuration management

**Cost Optimization**
- Filter unnecessary logs at source (forwarder)
- Implement index time extractions
- Archive old data to object storage
- Monitor license usage daily

---

## Further Learning

- **Splunk Documentation**: https://docs.splunk.com
- **Splunk Certification**: https://www.splunk.com/en_us/training.html
- **Docker Hub**: https://hub.docker.com/r/splunk/splunk
- **GitHub - Docker Splunk**: https://github.com/splunk/docker-splunk

---

---

# Splunk: Practical Examples & Code Reference

## Table of Contents
1. [Quick Start Commands](#quick-start)
2. [Common SPL Queries](#spl-queries)
3. [Configuration Examples](#config-examples)
4. [Docker Commands](#docker-commands)
5. [Forwarder Setup](#forwarder-setup)
6. [Real-World Use Cases](#real-world)

---

## Quick Start Commands {#quick-start}

### Check Splunk Status
```bash
# Check if Splunk is running
ps aux | grep splunk

# Check version
/opt/splunk/bin/splunk --version

# View server health
/opt/splunk/bin/splunk list licenses -auth admin:password

# Check index status
/opt/splunk/bin/splunk list indexes -auth admin:password
```

### CLI Operations
```bash
# Create index
/opt/splunk/bin/splunk add index myindex -auth admin:password

# Add forwarding server
/opt/splunk/bin/splunk add forward-server indexer1:9997 -auth admin:password

# Restart Splunk service
/opt/splunk/bin/splunk restart

# Stop Splunk
/opt/splunk/bin/splunk stop

# Show current config
/opt/splunk/bin/splunk show config outputs -auth admin:password
```

### Web API Operations
```bash
# Get server info
curl -k -u admin:password https://localhost:8089/services/server/info

# List indexes via REST API
curl -k -u admin:password https://localhost:8089/services/data/indexes

# Create saved search
curl -k -u admin:password -X POST \
  https://localhost:8089/services/saved/searches \
  -d "name=my_search&search=index%3Dmain"

# Get search job results
curl -k -u admin:password https://localhost:8089/services/search/jobs
```

---

## Common SPL Queries {#spl-queries}

### Basic Queries

**Search All Events in Main Index**
```spl
index=main
```

**Search Specific Time Range**
```spl
index=main earliest=-24h@h latest=now
```

**Search with Multiple Conditions**
```spl
index=main sourcetype=nginx:access status=500 host=web-01
```

**Case-Insensitive Search**
```spl
index=main [search client_ip=192.168.1.*]
```

### Field Extraction & Transformation

**Extract Fields on the Fly**
```spl
index=main
| rex field=_raw "user=(?<username>\w+)"
| rex field=_raw "action=(?<action>\w+)"
```

**Rename Fields**
```spl
index=main
| rename host as hostname, source as log_source
```

**Keep Only Specific Fields**
```spl
index=main
| fields host, user, status, response_time
```

**Lookup External Data**
```spl
index=main
| lookup users.csv user_id OUTPUT user_name, department
```

### Aggregation & Statistics

**Count Events by Host**
```spl
index=main
| stats count by host
```

**Average Response Time**
```spl
index=main
| stats avg(response_time) as avg_rt, max(response_time) as max_rt by host
```

**Multiple Aggregations**
```spl
index=main
| stats count, avg(latency), max(latency), min(latency) by host, status
```

**Time-Based Aggregation**
```spl
index=main
| timechart span=5m avg(response_time), count by host
```

**Top 10 Values**
```spl
index=main
| top 10 user
| rename count as User_Count, percent as Percentage
```

**Unique Values Count**
```spl
index=main
| stats dc(user) as unique_users, count by host
```

### Filtering & Conditional Logic

**Where Clause (Post-Search Filter)**
```spl
index=main
| stats count by host, status
| where status >= 400
```

**If/Then Logic**
```spl
index=main
| stats count as event_count by host
| eval severity=if(event_count > 1000, "CRITICAL", 
                    if(event_count > 500, "HIGH", "MEDIUM"))
```

**Multiple Conditions**
```spl
index=main
| eval priority=case(
    status >= 500, "P1",
    status >= 400, "P2",
    status >= 300, "P3",
    1=1, "P4"
  )
```

### Joins & Set Operations

**Inner Join on Field**
```spl
index=main | search status=200
| join type=inner host [search index=main status=500]
```

**Append Results**
```spl
[search index=main status=200] 
| append 
[search index=main status=500]
```

**Set Difference**
```spl
[search index=main sourcetype=web status=200 | fields host]
| diff
[search index=main sourcetype=app status=500 | fields host]
```

### Transactions & Session Analysis

**Group Related Events**
```spl
index=main
| transaction user_id timeout=30m
| stats avg(duration) as avg_session_duration by user_id
```

**Session Start Detection**
```spl
index=main
| transaction user_id startswith=eval(action="login") endswith=eval(action="logout")
| stats count as sessions, sum(duration) as total_time by user_id
```

### Time Series & Trending

**Daily Trend**
```spl
index=main
| timechart span=1d count as daily_events by status
```

**Hourly Comparison**
```spl
index=main
| timechart span=1h count, avg(response_time) by host
```

**Percent Change**
```spl
index=main
| timechart span=1h count as events
| eval percent_change=round(((events-previous_events)/previous_events)*100, 2)
```

### Anomaly Detection

**Standard Deviation-Based Alerting**
```spl
index=main
| stats avg(latency) as avg_latency, stdev(latency) as stdev_latency
| eval upper_bound=avg_latency+(2*stdev_latency)
| eval lower_bound=avg_latency-(2*stdev_latency)
| eval anomaly=if(latency > upper_bound OR latency < lower_bound, "YES", "NO")
```

**Predict Future Values**
```spl
index=main
| timechart span=1h count
| predict _time count future_timespan=24
```

### Security & Threat Detection

**Failed Login Tracking**
```spl
index=security sourcetype=auth failed_login=true
| stats count as failed_attempts by user, src_ip
| where failed_attempts > 5
| eval severity="HIGH"
```

**Lateral Movement Detection**
```spl
index=security sourcetype=windows:security EventCode=4625 OR EventCode=4624
| transaction user_name, host maxspan=1h
| where mvcount(host) > 3
| eval suspicious="YES"
```

**Data Exfiltration Detection**
```spl
index=network sourcetype=netflow bytes_out > 1000000
| stats sum(bytes_out) as total_bytes by src_ip, dst_ip
| where total_bytes > 10000000
| lookup malicious_ips.csv ip OUTPUT threat_level
```

---

## Configuration Examples {#config-examples}

### Inputs Configuration

**Monitor Multiple Sources**
```ini
# /opt/splunk/etc/system/local/inputs.conf

# Apache web server logs
[monitor:///var/log/apache2/access.log]
index = web_metrics
sourcetype = apache:access
source = /var/log/apache2/access.log
NO_BINARY_CHECK = true

# Application logs
[monitor:///var/log/myapp/*.log]
index = app_logs
sourcetype = app:json
source = /var/log/myapp
recursive = true

# System logs
[tcpin://:514]
connection_host = ip
sourcetype = syslog
index = system

# Python script to collect metrics
[script:///usr/local/bin/collect_metrics.py]
interval = 60
sourcetype = custom:metrics
index = metrics
source = script
```

### Outputs Configuration with Load Balancing

```ini
# /opt/splunk/etc/system/local/outputs.conf

[tcpout]
defaultGroup = indexers
forwardedindex.filter.disable = true
indexAndForward = false

[tcpout:indexers]
server = indexer1.prod.internal:9997, indexer2.prod.internal:9997, indexer3.prod.internal:9997
loadBalanceSetting = persistentVolumeBased
autoLBFrequency = 30
autoLBVolume = 10485760
autoLBFrequencyScalar = 60

[tcpout:indexers]
sslVerifyServerCert = true
sslCertPath = /opt/splunk/etc/apps/ssl/client.pem
sslPassword = $7$encrypted_password

maxConnectionsPerIndexer = 4
maxQueueSize = 512KB
tcpSendBufSz = 4096KB
```

### Props Configuration for Parsing

```ini
# /opt/splunk/etc/system/local/props.conf

# Apache Access Logs
[apache:access]
TRANSFORMS = parse_apache, enrich_ip_geolocation
REPORT-fields = apache_fields
LINE_BREAKER = (?m)^

# JSON Logs
[json_logs]
AUTO_KV_JSON = true
KV_MODE = json
TIMESTAMP_FIELDS = timestamp, @timestamp
LOOKUP-fields = enrichment_lookup

# CSV Logs
[csv_data]
DELIMS = ","
FIELDS = timestamp, user, action, status, message

# Multiline Logs (Java Stack Traces)
[java:stacktrace]
SHOULD_LINEMERGE = true
LINE_BREAKER = (^[a-zA-Z]|\s+at\s+)
LEARN_SOURCETYPE = false

# Custom Application
[myapp:log]
EXTRACT-method = method=(?<http_method>\w+)
EXTRACT-status = response=(?<status>\d{3})
EXTRACT-duration = duration=(?<response_time>\d+)ms
```

### Transforms Configuration

```ini
# /opt/splunk/etc/system/local/transforms.conf

# Parse Apache access log
[parse_apache]
REGEX = (?P<client_ip>[\w\.]+)\s+-\s+\[(?P<timestamp>[^\]]*)\]\s+"(?P<method>\w+)\s+(?P<uri>[^\s]*)\s+(?P<protocol>[^"]*)"\s+(?P<status>\d+)\s+(?P<bytes>\d+)
FORMAT = client_ip::$1 timestamp::$2 method::$3 uri::$4 protocol::$5 status::$6 bytes::$7

# Geolocation enrichment
[enrich_ip_geolocation]
filename = geoip.csv
min_matches = 1

# Route errors to specific index
[route_errors_to_security]
REGEX = ERROR|CRITICAL
DEST_KEY = _MetaData:Index
FORMAT = security_alerts

# Filter debug logs
[filter_debug_logs]
REGEX = DEBUG|TEST
DEST_KEY = queue
FORMAT = nullQueue
```

### Server Configuration for Clustering

```ini
# /opt/splunk/etc/system/local/server.conf

# License
[license]
master_uri_ = https://license-master.prod.internal:8089
connection_timeout = 20
send_to_phonehome = true

# General
[general]
pass4SymmKey = $7$YOUR_ENCRYPTED_KEY
server_ui = default
session_timeout = 1h

# SSL Configuration
[sslConfig]
enableSSL = true
sslVersions = tls1.2, tls1.3
serverCert = /opt/splunk/etc/apps/ssl/server.pem
requireClientCert = false
sslPassword = $7$encrypted_password
cipherSuite = ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384

# Indexer Clustering
[clustering]
mode = peer
master_uri = https://cluster-master.prod.internal:8089
pass4SymmKey = $7$CLUSTER_KEY
cxn_timeout = 30
rep_cxn_timeout = 30
replication_port = 8080
heartbeat_timeout = 60
replication_use_ssl = true

# Search Head Clustering
[shclustering]
disabled = false
pass4SymmKey = $7$SHC_KEY
shcluster_label = prod_shc
conf_deploy_fetch_url = https://deployment-server:8089
server_uri = https://search-head-1:8089
```

---

## Docker Commands {#docker-commands}

### Standalone Container

```bash
# Pull latest image
docker pull splunk/splunk:latest

# Run single instance
docker run -d \
  --name splunk-single \
  -p 8000:8000 \
  -p 9997:9997 \
  -e SPLUNK_START_ARGS='--accept-license' \
  -e SPLUNK_PASSWORD='YourSecurePass123!' \
  -e SPLUNK_GENERAL_TERMS='--accept-sgt-current-at-splunk-com' \
  splunk/splunk:latest

# Check logs
docker logs -f splunk-single

# Access container shell
docker exec -it splunk-single /bin/bash

# View Splunk logs inside container
docker exec splunk-single tail -f /opt/splunk/var/log/splunk/splunkd.log
```

### Multi-Instance with Docker Compose

```bash
# Start cluster
docker-compose up -d

# View service status
docker-compose ps

# View logs
docker-compose logs -f search-head

# Scale indexers
docker-compose up -d --scale indexer=5

# Stop all services
docker-compose down

# Stop and remove volumes
docker-compose down -v
```

### Volume Management

```bash
# Create named volume
docker volume create splunk-data

# List volumes
docker volume ls

# Inspect volume
docker volume inspect splunk-data

# Backup volume
docker run --rm \
  -v splunk-data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/splunk-backup.tar.gz /data

# Restore from backup
docker volume create splunk-data-restore

docker run --rm \
  -v splunk-data-restore:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/splunk-backup.tar.gz -C /data --strip-components=1
```

### Network Configuration

```bash
# Create custom network
docker network create splunk-network

# Run containers on same network
docker run -d \
  --name indexer \
  --network splunk-network \
  -p 9997:9997 \
  splunk/splunk:latest

docker run -d \
  --name search-head \
  --network splunk-network \
  -p 8000:8000 \
  splunk/splunk:latest

# Connect existing container
docker network connect splunk-network splunk-single

# Test connectivity
docker exec indexer ping search-head
```

### Build Custom Image

```dockerfile
# Dockerfile
FROM splunk/splunk:latest

USER root

# Install additional tools
RUN apt-get update && \
    apt-get install -y curl wget git && \
    rm -rf /var/lib/apt/lists/*

# Copy custom configs
COPY config/inputs.conf /opt/splunk/etc/system/local/
COPY config/props.conf /opt/splunk/etc/system/local/
COPY scripts/ /opt/splunk/etc/apps/custom/

USER splunk

# Build image
docker build -t my-splunk:custom .

# Run custom image
docker run -d \
  --name splunk-custom \
  -p 8000:8000 \
  -e SPLUNK_PASSWORD='Pass123!' \
  -e SPLUNK_START_ARGS='--accept-license' \
  my-splunk:custom
```

---

## Forwarder Setup {#forwarder-setup}

### Universal Forwarder Installation

**Linux**
```bash
# Download
wget -O splunk-forwarder.tgz \
  'https://www.splunk.com/bin/splunk/DownloadActivityServlet?architecture=x86_64&platform=Linux&version=9.0.0&wget=true'

# Extract
tar xzf splunk-forwarder.tgz -C /opt

# Create user
useradd -m splunk

# Set permissions
chown -R splunk:splunk /opt/splunk*

# Start forwarder
/opt/splunk/bin/splunk start --accept-license --answer-yes --no-prompt
```

**Windows**
```powershell
# Download MSI
$url = "https://www.splunk.com/bin/splunk/DownloadActivityServlet?architecture=x86_64&platform=windows&version=9.0.0&wget=true"
Invoke-WebRequest -Uri $url -OutFile splunk-forwarder.msi

# Install
msiexec.exe /i splunk-forwarder.msi ACCEPT_TERMS=1 /qn

# Start service
Start-Service SplunkForwarder
```

### Forwarder Configuration

**Basic inputs.conf**
```ini
[monitor:///var/log/app.log]
index = main
sourcetype = app:log
source = /var/log/app.log
```

**outputs.conf for Load Balancing**
```ini
[tcpout]
defaultGroup = indexers
forwardedindex.filter.disable = true

[tcpout:indexers]
server = indexer1:9997, indexer2:9997, indexer3:9997
loadBalanceSetting = persistentVolumeBased
autoLBFrequency = 30
```

**Docker UF Deployment**
```bash
docker run -d \
  --name splunk-uf \
  -e SPLUNK_START_ARGS='--accept-license' \
  -e SPLUNK_PASSWORD='Pass123!' \
  -e SPLUNK_FORWARD_SERVER='indexer:9997' \
  -e SPLUNK_ADD='unix' \
  -v /var/log:/host/var/log:ro \
  splunk/universalforwarder:latest
```

### Troubleshooting Forwarders

```bash
# Check forwarder status
/opt/splunk/bin/splunk list forward-server -auth admin:password

# View forwarder logs
tail -f /opt/splunk/var/log/splunk/metrics.log

# Test connectivity
telnet indexer-ip 9997

# Check queues
/opt/splunk/bin/splunk list inputstatus -auth admin:password

# Monitor forwarding
/opt/splunk/bin/splunk list inputs -auth admin:password
```

---

## Real-World Use Cases {#real-world}

### Use Case 1: Web Application Monitoring

**Setup**
```bash
# Install UF on web server
/opt/splunkforwarder/bin/splunk add forward-server indexer:9997 \
  -auth admin:password

# Configure to monitor Nginx
/opt/splunkforwarder/bin/splunk add monitor /var/log/nginx/ \
  -index web -sourcetype nginx:access \
  -auth admin:password
```

**Monitoring Query**
```spl
index=web sourcetype=nginx:access
| timechart span=5m count by status
| where status >= 400
```

### Use Case 2: Database Performance Monitoring

**DBX Setup (Splunk DB Connect)**
```ini
[my_database]
connection = mydb
query = SELECT * FROM sys.dm_exec_requests
sourcetype = mssql:perfmon
index = db_metrics
```

**Monitoring Query**
```spl
index=db_metrics sourcetype=mssql:perfmon
| stats avg(cpu_time) as avg_cpu, max(memory_usage) as peak_memory by session_id
| where avg_cpu > 1000
```

### Use Case 3: Cloud Monitoring (AWS)

**AWS Integration Setup**
```bash
# Install AWS Add-on
cd /opt/splunk/etc/apps
wget https://github.com/splunk/aws-cloudformation-templates/raw/master/addons/aws.tgz
tar xzf aws.tgz

# Configure AWS credentials
# Edit: /opt/splunk/etc/apps/Splunk_TA_aws/local/aws_inputs.conf
```

**Configuration**
```ini
[aws-cloudwatch-logs]
aws_account = default
aws_region = us-east-1
log_group_name = /aws/lambda/my-function
interval = 300
sourcetype = aws:cloudwatch
```

**Monitoring Query**
```spl
index=main sourcetype=aws:cloudwatch
| stats count as errors by msg_type, log_stream
| where errors > 10
```

### Use Case 4: Security Event Correlation

**Data Collection**
```ini
[monitor:///var/log/auth.log]
sourcetype = linux:auth
index = security

[tcpin://localhost:9514]
sourcetype = windows:security
index = security
```

**Alert Query**
```spl
index=security 
| transaction src_ip maxspan=5m 
| search eventtype=failed_login 
| stats count as failed_attempts by src_ip 
| where failed_attempts > 5
| alert_name "Brute Force Detected"
```

---

## Performance Tuning Commands

```bash
# Monitor current indexing rate
/opt/splunk/bin/splunk show inputstatus

# Check license usage
/opt/splunk/bin/splunk show license-usage -auth admin:password

# Optimize index
/opt/splunk/bin/splunk optimize datamodel

# Clear search cache
/opt/splunk/bin/splunk clean cache

# Monitor search load
/opt/splunk/bin/splunk list jobs auth=admin:password
```

---

## Useful Links

- **Splunk Docs**: https://docs.splunk.com
- **SPL Search Tutorial**: https://docs.splunk.com/Documentation/Splunk/latest/SearchTutorial/WelcometotheSearchTutorial
- **Props & Transforms**: https://docs.splunk.com/Documentation/Splunk/latest/Admin/Propsconf
- **Splunk Community**: https://community.splunk.com
- **Docker Hub**: https://hub.docker.com/r/splunk/splunk
