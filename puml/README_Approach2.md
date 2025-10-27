# Observability Strategy (Approach 2: Open Source Stack)

This document outlines an observability, logging, and alerting strategy for Yeet the casino platform, built on a flexible, vendor-neutral, and open-source-first toolset.

##  Guiding Philosophy

This approach prioritizes **power, flexibility, and a unified experience** over a single-vendor solution. By using best-in-class tools for each pillar of observability, we gain deep insights and avoid vendor lock-in.

The core of this stack is built on:
* **Prometheus** for metrics.
* **Loki** for logs.
* **Tempo** for traces.
* **OpenTelemetry (OTel)** as the single standard for collecting and forwarding all telemetry.
* **Grafana** as the unified "single pane of glass" for visualization and correlation.


## Architecture

The central component is the **OpenTelemetry Collector**, which acts as the single "agent" for all services. Applications are instrumented once (with the OTel SDK) and send all their data to the Collector. The Collector then intelligently processes this data and exports it to the correct open-source backend (Prometheus, Loki, Tempo).



## Core Pillars

### 1. The Collector: OpenTelemetry (OTel)
The **OTel Collector** is the heart of this architecture. It's a single agent (run as a sidecar or service in ECS) that receives all telemetry data from our applications.
* **Receive:** Applications (`BANK`, `REWARDS`, `SOCKETS`) are instrumented with the OTel SDK and send all data to the Collector.
* **Process:** The Collector can batch, filter, or enrich data (e.g., add ECS metadata like `task_id`).
* **Export:** It then exports the data to the correct backend:
    * Metrics -> **Prometheus**
    * Logs -> **Loki**
    * Traces -> **Tempo**

This simplifies application code (it only talks to OTel) and gives us a central point of control.

### 2. Metrics: Prometheus
* **Service:** **Prometheus**
* **How it Works:**
    * **Application Metrics:** Receives application metrics (like `bet_processed_total`) from the OTel Collector.
    * **Infrastructure Metrics:** For AWS services, we use exporters. Prometheus will "scrape":
        * **`CloudWatch Exporter`**: To pull metrics for RDS, ALB, and MSK (like `ConsumerLag`).
        * **`Blackbox Exporter`**: For uptime monitoring. This exporter will "probe" our ALB and `SOCKETS` endpoints. Prometheus scrapes the probe's *result* (e.g., `probe_success`).

### 3. Logs: Loki
* **Service:** **Grafana Loki**
* **How it Works:**
    * Loki is a log aggregation system "like Prometheus, but for logs." It's highly efficient and cost-effective.
    * **Application Logs:** The OTel Collector forwards all application (JSON) logs to Loki.
    * **Infrastructure Logs:** A separate agent like **Promtail** can be used to ship other logs (e.g., ALB access logs from S3) to Loki.

### 4. Traces: Tempo
* **Service:** **Grafana Tempo**
* **How it Works:**
    * Tempo is a high-volume, minimal-dependency distributed tracing backend.
    * The OTel Collector forwards all traces to Tempo.
    * Tempo's primary job is to store traces and retrieve them by their ID, integrating directly with Grafana for visualization.

### 5. Visualization: Grafana
* **Service:** **Grafana**
* **How it Works:** This is the unified UI for everything. We will add three data sources to Grafana:
    1.  **Prometheus** (for metrics)
    2.  **Loki** (for logs)
    3.  **Tempo** (for traces)
* **Correlation:** Grafana's key feature is its ability to link these. From a metric spike on a dashboard (Prometheus), you can instantly jump to the logs (Loki) or traces (Tempo) *from that exact time period*, providing a seamless debugging workflow.

### 6. Alerting: Alertmanager
* **Service:** **Alertmanager**
* **How it Works:**
    * Prometheus defines alert rules using its query language (PromQL) (e.g., `ConsumerLag > 1000`).
    * If a rule is met, it "fires" an alert to **Alertmanager**.
    * Alertmanager handles deduplicating, grouping, silencing, and routing alerts to the correct destination (e.g., **PagerDuty** for critical alerts, **Slack** for warnings).


## Addressing Mission-Critical Areas

| Requirement | Solution |
| :--- | :--- |
| **Bets Processed Correctly** | **Metric (Prometheus):** `bet_processed_total{status="success"}` & `bet_processed_total{status="failed"}`. <br/> **Alert (Alertmanager):** `rate(bet_processed_total{status="failed"}[5m]) > 0`. <br/> **Debug:** Correlate metric spikes in Grafana with logs in Loki (filtered by `event="bet_failed"`) and traces in Tempo. |
| **Rewards Calculated Properly** | **Metric (Prometheus):** `aws_msk_consumergroup_lag_sum` (from `CloudWatch Exporter`). <br/> **Alert (Alertmanager):** `aws_msk_consumergroup_lag_sum{consumergroup="rewards-service"} > 1000 FOR 5m`. |
| **User Abuse / Scraping** | **AWS WAF:** Use rate-limiting rules (still the best tool for prevention). <br/> **Logs (Loki):** Ship WAF/ALB logs (via Promtail) to Loki. <br/> **Dashboard (Grafana):** Create a dashboard of `count_over_time({job="alb_logs"} \| status_code="429" [1m]) by ip`. |
| **Deposits & Withdrawals** | **Metric (Prometheus):** `deposit_total{status="failed"}`. <br/> **Alert (Alertmanager):** **HIGH PRIORITY** `PagerDuty` alert if `rate(deposit_total{status="failed"}[2m]) > 0`. |
| **Uptime (e.g., Chat)** | **Metric (Prometheus):** `probe_success{job="chat-websocket"}` (from `Blackbox Exporter`). <br/> **Alert (Alertmanager):** `probe_success{job="chat-websocket"} == 0` (Alert if the probe fails). |
| **Casino Admin View** | **Dashboard (Grafana):** A non-technical dashboard showing key business metrics from Prometheus: `active_users` (from a `SOCKETS` gauge), `total_deposits` (from a counter), etc. |