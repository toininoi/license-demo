# 📊 TIBCO Monitoring: FTL, EMS, RV, and BW

This guide details how to expose, scrape, and visualize Prometheus metrics across a containerized TIBCO stack, including Messaging (FTL, EMS, RV) and Cloud Software Group BusinessWorks Container Edition (BWCE).

## 🏗️ Architecture & Port Mapping

The diagram below shows the monitoring flow. Messaging components are **scraped** directly by Prometheus, while BWCE **pushes** metrics to the OTel Collector via gRPC, which is then scraped by Prometheus.

```mermaid
%%{init: {
  'theme': 'base', 
  'look': 'handDrawn', 
  'themeVariables': { 
    'fontFamily': 'Comic Sans MS, cursive',
    'primaryColor': '#ffffff',
    'mainBkg': '#ffffff',
    'lineColor': '#444444'
  }
}}%%

%%{init: {'theme': 'neutral'}}%%
graph RL
    User["👤 User (Web Browser)"]

    subgraph DockerHost[Docker Host]
        subgraph DockerNetwork[tibco-network]
            
            %% Monitoring Stack
            subgraph Monitoring[Monitoring Stack]
                Grafana["Grafana<br/>Port: 3000"]
                Prometheus["Prometheus<br/>Port: 9090"]
                OTEL["OTel Collector<br/>gRPC: 4317 | Scrape: 8889"]
            end

            %% BWCE
            subgraph BW Container[ BW Container]
                BWCE["bwce_service<br/>(OTLP Push)"]
            end

            %% TIBCO FTL / EMS Cluster
            subgraph TibcoCluster[TIBCO FTL & EMS Cluster]
                FTL1["ftlserver1<br/>FTL: 8585 | EMS: 7220"]
                FTL2["ftlserver2<br/>FTL: 8585 | EMS: 7220"]
                FTL3["ftlserver3<br/>FTL: 8585 | EMS: 7220"]
            end

            %% TIBCO Rendezvous
            subgraph TibcoRV[TIBCO Rendezvous]
                RVL["rv-listener<br/>RVD HTTP: 7580"]
                RVS["rv-sender<br/>RVD HTTP: 7580"]
            end
            
        end
    end

    %% Connections
    User -->|http://localhost:3000| Grafana
    Grafana -->|Queries| Prometheus
    
    %% Push Flow
    BWCE -->|OTLP gRPC:4317| OTEL
    
    %% Pull/Scrape Flow
    Prometheus -.->|Scrapes :8889| OTEL
    Prometheus -.->|Scrapes :8585/7220| FTL1
    Prometheus -.->|Scrapes :8585/7220| FTL2
    Prometheus -.->|Scrapes :8585/7220| FTL3
    Prometheus -.->|Scrapes :7580| RVL
    Prometheus -.->|Scrapes :7580| RVS

    %% Styling
    style DockerHost fill:#f0f8ff,stroke:#2563eb,stroke-width:2px
    style DockerNetwork fill:#f0fdf4,stroke:#16a34a,stroke-width:2px,stroke-dasharray: 5 5
    style User fill:#a2d2ff,stroke:#0056b3,stroke-width:2px,rx:10,ry:10,color:#000
    style Grafana fill:#d8f3dc,stroke:#2d6a4f,stroke-width:2px,rx:10,ry:10,color:#000
    style Prometheus fill:#f96,stroke:#a33b00,stroke-width:3px,color:#000,font-weight:bold
    style OTEL fill:#ffb703,stroke:#e85d04,stroke-width:2px,rx:10,ry:10,color:#000
    style BWCE fill:#e6ccb2,stroke:#a68a64,stroke-width:2px,rx:10,ry:10,color:#000
    style FTL1 fill:#ffcccc,stroke:#cc0000,stroke-width:2px,rx:10,ry:10,color:#000
    style FTL2 fill:#ffcccc,stroke:#cc0000,stroke-width:2px,rx:10,ry:10,color:#000
    style FTL3 fill:#ffcccc,stroke:#cc0000,stroke-width:2px,rx:10,ry:10,color:#000
    style RVL fill:#9f9,stroke:#008000,stroke-width:3px,color:#000,font-weight:bold
    style RVS fill:#9f9,stroke:#008000,stroke-width:3px,color:#000,font-weight:bold   
```

The monitoring flow is designed so that Messaging components are **scraped** directly by Prometheus, while BWCE **pushes** metrics to the OTel Collector via gRPC, which is then scraped by Prometheus.

## 🔌 1. Exposing `/metrics` Configuration

### ⚙️ TIBCO BWCE (OTEL Collector)
BWCE uses the OpenTelemetry (OTLP) gRPC exporter to push metrics to a central collector.
* **Environment Variable:**
  BW_JAVA_OPTS=-Dbw.engine.opentelemetry.enable=true -Dbw.engine.opentelemetry.metric.enable=true -Dbw.engine.opentelemetry.metric.exporter.endpoint=http://otel_collector:4317
  
* **Collector Logic:** The OTel Collector receives gRPC traffic on `4317` and exposes an HTTP scrape endpoint for Prometheus on port `8889`.

### ✉️ TIBCO EMS (10.4+)
EMS metrics are exposed via the monitor port.
* **Flag:** `-monitor_listen http://<hostname>:7220` (Exposed as standard Prometheus text format).
* **Compose Port:** `7220`

### ⚡ TIBCO FTL (7.2+)
Natively exposed on the Realm Server port. No extra configuration required beyond ensuring connectivity.
* **Compose Port:** `8585`

### 📡 TIBCO Rendezvous (RV 9.0+)
Metrics are exposed via the `rvd` HTTP administration interface.
* **Flag:** `rvd -http 7580 &`
* **Compose Port:** `7580`

---

## 📈 2. Dashboard PromQL Rules

Below the PromQL for all products

### ✉️ EMS/FTL/RV/BW Expiration
| Component | PromQL Query |
| :--- | :--- |
| **EMS** | max by (instance) (tibco_ems_server_license_expiration_seconds) |
| **FTL** | max by (instance) (tibco_ftl_server_license_expiration_seconds) |
| **RV** | max by (instance) (tibco_rv_lease_expiration_seconds) |
| **BW** | app_metrics_com_tibco_bw_license_expiration_seconds / 86400 |

---

## 🧪 3. Running the Stack

### ▶️ Start all containers
Ensure your `prometheus.yml` and `otelcol-config.yaml` are present in the directory.
```bash
docker-compose up -d
```

![Screen](./img/dashboard.png)

### 🛑 Stop all containers
```bash
docker-compose down
```

