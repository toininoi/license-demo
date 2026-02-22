# 📊 TIBCO Messaging Monitoring: FTL, EMS, and RV

This guide details how to expose, scrape, and visualize Prometheus metrics across a containerized TIBCO messaging stack (FTL, EMS, and Rendezvous), with a specific focus on tracking software license expiration times.



## 🏗️ Architecture & Port Mapping

The following diagram illustrates how the Docker containers communicate and how Prometheus scrapes the metrics from each specific port within the Docker network.

```mermaid
graph LR
    User["👤 User (Web Browser)"]

    subgraph DockerHost[Docker Host]
        subgraph DockerNetwork[tibco-network]
            
            %% Monitoring Stack
            subgraph Monitoring[Monitoring Stack]
                Grafana["Grafana<br/>Internal: 3000 | Exposed: 3000"]
                Prometheus["Prometheus<br/>Port: 9090"]
            end

            %% TIBCO FTL / EMS Cluster
            subgraph TibcoCluster[TIBCO FTL & EMS Cluster]
                FTL1["ftlserver1<br/>FTL :8585 | EMS :7220"]
                FTL2["ftlserver2<br/>FTL :8585 | EMS :7220"]
                FTL3["ftlserver3<br/>FTL :8585 | EMS :7220"]
            end

            %% TIBCO Rendezvous
            subgraph TibcoRV[TIBCO Rendezvous]
                RVL["rv-listener<br/>RVD HTTP :7580"]
                RVS["rv-sender<br/>RVD HTTP :7580"]
            end
            
        end
    end

    %% Connections
    User -->|http://localhost:3000| Grafana
    Grafana -->|Queries| Prometheus
    Prometheus -.->|Scrapes /metrics| FTL1
    Prometheus -.->|Scrapes /metrics| FTL2
    Prometheus -.->|Scrapes /metrics| FTL3
    Prometheus -.->|Scrapes /metrics| RVL
    Prometheus -.->|Scrapes /metrics| RVS

    %% Styling for Subgraphs
    style DockerHost fill:#f0f8ff,stroke:#2563eb,stroke-width:2px,color:#1e293b
    style DockerNetwork fill:#f0fdf4,stroke:#16a34a,stroke-width:2px,stroke-dasharray: 5 5,color:#14532d
```

## 🔌 1. Exposing `/metrics` in Docker Compose

Each TIBCO component handles Prometheus metrics slightly differently. To expose them, your `docker-compose.yml` must configure the correct flags and ports for each service.

### ✉️ TIBCO EMS (10.4+)
EMS exposes its metrics via the monitor port.
* **Flag:** `-monitor_listen http://<hostname>:7220` (Configured in your `tibftlserver_cluster.yaml` under the `tibemsd` section).
* **Compose Port:** `7220`

### ⚡ TIBCO FTL (7.2+)
FTL natively exposes Prometheus metrics on its Realm Server port. No special flags are needed, just ensure the port is accessible.
* **Compose Port:** `8585`

### 📡 TIBCO Rendezvous (RV 8.8+)
RV exposes metrics via the Rendezvous Daemon (`rvd`) HTTP administration interface. Client tools (`tibrvsend`/`tibrvlisten`) do not expose metrics directly; you must start the daemon explicitly with the HTTP flag.
* **Flag:** Start the daemon using `rvd -http 7580 &` before running your client commands.
* **Compose Port:** `7580`

**Example RV Sender Compose Command:**
```yaml
command: >
  bash -c "rvd -listen tcp:7500 -http 7580 & sleep 5; while true; do tibrvsend -service 7500 -network ';' -daemon tcp:7500 TEST.SUBJECT 'Ping'; sleep 10; done"
```

## 🧪 2. Test docker compose 

### 📋 Requirements 
* EMS,FTL and RV images built
* FTL yaml and EMS json config files 
* Valid and not expired license file

### ▶️ Start all containers
```bash
docker-compose -f docker-compose.yml up -d
```


*Note: `license-dashboard.json` is delivered as a sample to be imported to Grafana after the start of all containers.*



![Screen](./img/dashboard.png)

### 🛑 Stop all containers
```bash
docker-compose -f docker-compose.yml down
```
