# Elasticsearch Cluster Setup

## Step-by-Step Guide: Deploying a Multi-Node Elasticsearch & Kibana Cluster with Filebeat on Debian

This guide documents the deployment of a three-node Elasticsearch cluster, Kibana, and Filebeat using Docker on Debian.

> **Security note:** The original setup document contains example passwords. They are represented here as placeholders so credentials are not published in a GitHub repository. Store real credentials in `.env` or another secure secret-management mechanism and never commit them.

---

## 1. Prerequisites & System Preparation

Before beginning, ensure your Debian system satisfies the following minimum prerequisites:

- **OS:** Debian 11 (Bullseye) or Debian 12 (Bookworm)
- **CPU:** 4+ vCPUs recommended
- **RAM:** 8 GB minimum; 12 GB+ recommended for 3 Elasticsearch nodes + Kibana
- **Storage:** 30 GB+ free disk space
- **Privileges:** root or a user with sudo access

---

## Step 2: Installing Docker & Docker Compose on Debian

We install the official Docker Engine package directly from Docker's repository.

### 2.1 Update Package Index and Install Dependencies

```bash
sudo apt-get update && sudo apt-get install -y \
  ca-certificates \
  curl \
  gnupg \
  lsb-release
```

**Why:** `ca-certificates` allows HTTPS connections to repositories; `curl` downloads GPG keys; and `gnupg` handles cryptographic verification.

### 2.2 Add Docker's Official GPG Key

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

**Why:** Verifies that Docker packages are authentic and untampered.

### 2.3 Add Docker Repository to APT Sources

```bash
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/debian \
$(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 2.4 Install Docker Engine and Docker Compose Plugin

```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 2.5 Enable and Verify Docker

```bash
sudo systemctl enable --now docker
sudo docker --version
sudo docker compose version
```

---

## Step 3: Configuring System Requirements

Elasticsearch uses a `mmapfs` directory by default to store its indices. The default Linux memory-map limit may be too low.

### 3.1 Increase Virtual Memory Allocation

Set `vm.max_map_count` to at least `262144`:

```bash
sudo sysctl -w vm.max_map_count=262144
```

This prevents errors such as:

```
max virtual memory areas vm.max_map_count [65530] is too low
```

### 3.2 Make the Setting Persistent

```bash
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

## Step 4: Setting Up the Project Directory & .env Configuration

Create a dedicated project directory for certificates, Filebeat configuration, environment variables, and Docker definitions.

### 4.1 Create Directory Hierarchy

```bash
mkdir -p ~/elastic-cluster/{certs,filebeat}
cd ~/elastic-cluster
```

Directory purposes:

- `certs/` — TLS certificates shared across nodes
- `filebeat/` — Filebeat configuration files

### 4.2 Create the .env File

Create `~/elastic-cluster/.env`:

```dotenv
# Security Passwords
ELASTIC_PASSWORD=<SET_A_STRONG_ELASTIC_PASSWORD>
KIBANA_PASSWORD=<SET_A_STRONG_KIBANA_PASSWORD>

# Elastic Stack Version
STACK_VERSION=8.12.2

# Cluster Port Bindings
ES_PORT=9200
KIBANA_PORT=5601

# JVM Memory Allocation (per node)
MEM_LIMIT=2073741824

# Security / Certificate Settings
CLUSTER_NAME=es-debian-cluster
LICENSE=basic
```

**Why:** Separates credentials, resource limits, and version definitions from the container orchestration configuration.

> **Important:** Do not commit `.env` to a public repository if it contains real credentials.

Recommended `.gitignore` entry:

```gitignore
.env
certs/
```

---

## Step 5: Generating TLS/SSL Certificates

Elasticsearch 8.x uses TLS for secure internal node-to-node communication and HTTP communication.

We use `elasticsearch-certutil` inside a temporary Docker container to generate a Certificate Authority and certificates for the nodes.

### 5.1 Define Certificate Generation Configuration

Create `instances.yml`:

```yaml
instances:
  - name: es01
    dns:
      - es01
      - localhost
    ip:
      - 127.0.0.1

  - name: es02
    dns:
      - es02
      - localhost
    ip:
      - 127.0.0.1

  - name: es03
    dns:
      - es03
      - localhost
    ip:
      - 127.0.0.1
```

This defines the Subject Alternative Names (SANs) used by each node.

### 5.2 Generate Certificates Using an Ephemeral Container

```bash
docker run --rm \
  -v $(pwd):/usr/share/elasticsearch/config/cert_gen \
  --workdir /usr/share/elasticsearch/config/cert_gen \
  docker.elastic.co/elasticsearch/elasticsearch:8.12.2 \
  bash -c "bin/elasticsearch-certutil ca --silent --pem -out ca.zip && \
  unzip ca.zip && \
  bin/elasticsearch-certutil cert --silent --pem --ca-cert ca/ca.crt --ca-key ca/ca.key --in instances.yml -out certs.zip && \
  unzip certs.zip"
```

### 5.3 Move Certificates and Set Permissions

```bash
mv ca es01 es02 es03 certs/
sudo chown -R 1000:1000 certs/
chmod -R 750 certs/
```

The Elasticsearch Docker image uses UID 1000 for the Elasticsearch user, so the ownership and permissions allow the container to read the TLS material.

---

## Step 6: Drafting the Multi-Node docker-compose.yml

Create `docker-compose.yml` containing:

- Elasticsearch node `es01`
- Elasticsearch node `es02`
- Elasticsearch node `es03`
- Kibana
- Persistent Docker volumes
- A shared Docker network

```yaml
version: "3.8"

networks:
  elastic:
    driver: bridge

volumes:
  esdata01:
    driver: local
  esdata02:
    driver: local
  esdata03:
    driver: local
  kibanadata:
    driver: local

services:

  es01:
    image: docker.elastic.co/elasticsearch/elasticsearch:${STACK_VERSION}
    container_name: es01
    labels:
      co.elastic.logs/enabled: "true"
    volumes:
      - ./certs:/usr/share/elasticsearch/config/certs
      - esdata01:/usr/share/elasticsearch/data
    ports:
      - "${ES_PORT}:9200"
    environment:
      - node.name=es01
      - cluster.name=${CLUSTER_NAME}
      - cluster.initial_master_nodes=es01,es02,es03
      - discovery.seed_hosts=es02,es03
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
      - bootstrap.memory_lock=true
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=true
      - xpack.security.http.ssl.key=certs/es01/es01.key
      - xpack.security.http.ssl.certificate=certs/es01/es01.crt
      - xpack.security.http.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.key=certs/es01/es01.key
      - xpack.security.transport.ssl.certificate=certs/es01/es01.crt
      - xpack.security.transport.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.transport.ssl.verification_mode=certificate
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    ulimits:
      memlock:
        soft: -1
        hard: -1
    networks:
      - elastic
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -s -k https://localhost:9200 | grep -q 'missing authentication credentials'"
        ]
      interval: 10s
      timeout: 10s
      retries: 12

  es02:
    image: docker.elastic.co/elasticsearch/elasticsearch:${STACK_VERSION}
    container_name: es02
    labels:
      co.elastic.logs/enabled: "true"
    volumes:
      - ./certs:/usr/share/elasticsearch/config/certs
      - esdata02:/usr/share/elasticsearch/data
    environment:
      - node.name=es02
      - cluster.name=${CLUSTER_NAME}
      - cluster.initial_master_nodes=es01,es02,es03
      - discovery.seed_hosts=es01,es03
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
      - bootstrap.memory_lock=true
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=true
      - xpack.security.http.ssl.key=certs/es02/es02.key
      - xpack.security.http.ssl.certificate=certs/es02/es02.crt
      - xpack.security.http.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.key=certs/es02/es02.key
      - xpack.security.transport.ssl.certificate=certs/es02/es02.crt
      - xpack.security.transport.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.transport.ssl.verification_mode=certificate
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    ulimits:
      memlock:
        soft: -1
        hard: -1
    networks:
      - elastic
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -s -k https://localhost:9200 | grep -q 'missing authentication credentials'"
        ]
      interval: 10s
      timeout: 10s
      retries: 12

  es03:
    image: docker.elastic.co/elasticsearch/elasticsearch:${STACK_VERSION}
    container_name: es03
    labels:
      co.elastic.logs/enabled: "true"
    volumes:
      - ./certs:/usr/share/elasticsearch/config/certs
      - esdata03:/usr/share/elasticsearch/data
    environment:
      - node.name=es03
      - cluster.name=${CLUSTER_NAME}
      - cluster.initial_master_nodes=es01,es02,es03
      - discovery.seed_hosts=es01,es02
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
      - bootstrap.memory_lock=true
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=true
      - xpack.security.http.ssl.key=certs/es03/es03.key
      - xpack.security.http.ssl.certificate=certs/es03/es03.crt
      - xpack.security.http.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.key=certs/es03/es03.key
      - xpack.security.transport.ssl.certificate=certs/es03/es03.crt
      - xpack.security.transport.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.transport.ssl.verification_mode=certificate
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    ulimits:
      memlock:
        soft: -1
        hard: -1
    networks:
      - elastic
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -s -k https://localhost:9200 | grep -q 'missing authentication credentials'"
        ]
      interval: 10s
      timeout: 10s
      retries: 12

  kibana:
    image: docker.elastic.co/kibana/kibana:${STACK_VERSION}
    container_name: kibana
    depends_on:
      es01:
        condition: service_healthy
      es02:
        condition: service_healthy
      es03:
        condition: service_healthy
    volumes:
      - ./certs:/usr/share/kibana/config/certs
      - kibanadata:/usr/share/kibana/data
    ports:
      - "${KIBANA_PORT}:5601"
    environment:
      - SERVERNAME=kibana
      - ELASTICSEARCH_HOSTS=https://es01:9200
      - ELASTICSEARCH_USERNAME=kibana_system
      - ELASTICSEARCH_PASSWORD=${KIBANA_PASSWORD}
      - ELASTICSEARCH_SSL_CERTIFICATEAUTHORITIES=config/certs/ca/ca.crt
    networks:
      - elastic
```

### Docker Compose Directives

| Directive | Purpose |
|---|---|
| `cluster.initial_master_nodes` | Defines the master-eligible nodes involved in initial cluster bootstrapping. |
| `discovery.seed_hosts` | Specifies hostname endpoints used for node discovery. |
| `bootstrap.memory_lock=true` | Locks process memory to prevent Elasticsearch memory swapping. |
| `ES_JAVA_OPTS=-Xms1g -Xmx1g` | Sets the JVM heap minimum and maximum size. |

---

## Step 7: Bootstrapping and Starting the Elastic Stack

### 7.1 Launch Elasticsearch Nodes

```bash
docker compose up -d es01 es02 es03
```

This starts the core cluster services without waiting for Kibana.

### 7.2 Set the kibana_system Password

Once the cluster reaches healthy status:

```bash
docker exec -it es01 bash -c "
bin/elasticsearch-reset-password -u kibana_system -b -p <SET_KIBANA_SYSTEM_PASSWORD>
"
```

Kibana uses the built-in `kibana_system` service account to access Elasticsearch.

### 7.3 Start Kibana

```bash
docker compose up -d kibana
```

---

## Step 8: Verifying Cluster Health and Accessing Kibana

### 8.1 Check Cluster Health

```bash
curl --cacert certs/ca/ca.crt \
  -u elastic:<ELASTIC_PASSWORD> \
  https://localhost:9200/_cluster/health?pretty
```

Expected result:

```json
{
  "cluster_name" : "es-debian-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 3,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 0,
  "active_shards" : 0,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0
}
```

### 8.2 Access Kibana

Open:

```
http://<YOUR_DEBIAN_IP>:5601
```

Use the Elasticsearch `elastic` account and the password configured in your environment.

---

## Step 9: Configuring and Ingesting Logs with Filebeat

The guide deploys Filebeat on the Debian host to forward local system logs such as:

- `/var/log/auth.log`
- `/var/log/syslog`

### 9.1 Create Filebeat Configuration

Create `filebeat/filebeat.yml`:

```yaml
filebeat.inputs:
  - type: filestream
    id: debian-system-logs
    enabled: true
    paths:
      - /var/log/syslog
      - /var/log/auth.log

setup.kibana:
  host: "http://kibana:5601"

output.elasticsearch:
  hosts: ["https://es01:9200"]
  username: "elastic"
  password: "<ELASTIC_PASSWORD>"
  ssl.certificate_authorities: ["/usr/share/filebeat/certs/ca/ca.crt"]
  ssl.verification_mode: "full"

logging.level: info
```

### 9.2 Set Filebeat Permissions

Filebeat requires restrictive permissions on its configuration file:

```bash
sudo chown root:root filebeat/filebeat.yml
sudo chmod 600 filebeat/filebeat.yml
```

### 9.3 Add Filebeat to docker-compose.yml

Add the following service:

```yaml
  filebeat:
    image: docker.elastic.co/beats/filebeat:${STACK_VERSION}
    container_name: filebeat
    user: root
    networks:
      - elastic
    depends_on:
      es01:
        condition: service_healthy
      kibana:
        condition: service_started
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - ./certs:/usr/share/filebeat/certs:ro
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
```

### 9.4 Start Filebeat and Set Up Dashboards

Start Filebeat:

```bash
docker compose up -d filebeat
```

Initialize Filebeat pipelines and index templates:

```bash
docker exec -it filebeat filebeat setup -e
```

This loads Filebeat dashboards, ILM policies, and ingest pipelines into Kibana and Elasticsearch.

### 9.5 Verify Log Ingestion in Kibana

1. Open Kibana at `http://<YOUR_DEBIAN_IP>:5601`.
2. Navigate to **Management > Stack Management > Data Views**.
3. Create a data view using the pattern `filebeat-*`.
4. Select `@timestamp` as the timestamp field.
5. Go to **Discover** to view incoming system logs.

---

## Troubleshooting & Maintenance

| Issue | Cause | Solution |
|---|---|---|
| `max virtual memory areas vm.max_map_count [65530] is too low` | Host `vm.max_map_count` setting is too low. | Run `sudo sysctl -w vm.max_map_count=262144`. |
| Elasticsearch node exits with code 137 | Out-of-memory (OOM) kill by the kernel. | Increase Docker memory or adjust `-Xms1g -Xmx1g` in `docker-compose.yml`. |
| Certificate validation failed | Hostname mismatch or incorrect CA certificate path. | Check `instances.yml` and verify certificate paths. |
| Filebeat configuration file is writable | Insecure file permissions on `filebeat.yml`. | Run `sudo chown root:root filebeat.yml && sudo chmod 600 filebeat.yml`. |

---

## Operational Commands Quick Reference

### View Real-Time Cluster Logs

```bash
docker compose logs -f
```

### Check Running Container Status

```bash
docker compose ps
```

### Stop the Entire Cluster

```bash
docker compose down
```

### Stop the Cluster and Purge Stored Data Volumes

> **Warning:** This removes the persistent Docker volumes and stored Elasticsearch data.

```bash
docker compose down -v
```

---

## Architecture Overview

The deployment consists of:

```
                    ┌─────────────────────┐
                    │       Kibana        │
                    │       :5601         │
                    └──────────┬──────────┘
                               │
                         HTTPS / TLS
                               │
              ┌────────────────┴────────────────┐
              │       Docker Elastic Network   │
              └────────────────┬────────────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
   ┌────▼────┐            ┌────▼────┐            ┌────▼────┐
   │  es01   │            │  es02   │            │  es03   │
   │ :9200   │            │         │            │         │
   └────┬────┘            └────┬────┘            └────┬────┘
        │                      │                      │
        └──────────────────────┼──────────────────────┘
                               │
                          Elasticsearch
                             Cluster
                               ▲
                               │
                         ┌─────┴─────┐
                         │  Filebeat │
                         │   Logs    │
                         └───────────┘
```

---

## Project Structure

A typical project directory looks like:

```text
elastic-cluster/
├── .env
├── docker-compose.yml
├── instances.yml
├── certs/
│   ├── ca/
│   ├── es01/
│   ├── es02/
│   └── es03/
└── filebeat/
    └── filebeat.yml
```

For a public GitHub repository, keep sensitive files out of version control:

```gitignore
.env
certs/
```
