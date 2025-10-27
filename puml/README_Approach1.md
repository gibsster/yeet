# Observability Strategy (Approach 1: AWS Native)

This document outlines an observability, logging, and alerting strategy for the `BANK`, `REWARDS`, and `SOCKETS` microservices, using a tightly integrated, AWS-native toolset.

## Guiding Philosophy

The core principle of this approach is **simplicity and integration**. By using the AWS ecosystem (CloudWatch, X-Ray), we reduce the number of vendors, simplify IAM roles and billing, and leverage services that are designed to work together out-of-the-box with ECS, ALB, and MSK.


## Architecture

The architecture centralizes all telemetry into **Amazon CloudWatch**, which acts as the single hub for logs, metrics, traces, dashboards, and alarms.



## Core Pillars

### 1. Logging

* **Service:** **Amazon CloudWatch Logs**
* **How it Works:**
    * **Collection:** We use **AWS FireLens** (a simple configuration in your ECS task definition) to automatically collect all `stdout`/`stderr` logs from your microservices.
    * **Format:** Your applications MUST log in **structured JSON**. This is not optional.
        * **Bad:** `User 123 failed to bet.`
        * **Good:** `{"level": "WARN", "userId": "123", "event": "bet_failed", "reason": "insufficient_funds"}`
    * **Querying:** We use **CloudWatch Logs Insights** to run fast, SQL-like queries on our JSON logs (e.g., `stats count() by userId, reason | filter event = 'bet_failed'`).
* **WAF/ALB Logs:** We also send ALB and WAF access logs to CloudWatch (or S3) for security and abuse analysis.

### 2. Metrics

* **Service:** **Amazon CloudWatch Metrics**
* **How it Works:**
    * **Infrastructure Metrics:** CloudWatch provides these automatically. We get CPU/Memory for ECS, Request Count/5xx errors for ALB, and Consumer Lag for MSK.
    * **Application Metrics:** We use the **CloudWatch Embedded Metric Format (EMF)**. This is a simple but powerful technique:
        1.  Your app writes a special JSON log to `stdout` (e.g., `{"_aws": {"Metrics": [{"Name": "BetSuccess", "Unit": "Count"}]}, "betAmount": 100, "userId": "123"}`).
        2.  CloudWatch *automatically* sees this, creates a custom metric called `BetSuccess`, *and* stores the full log for context.
    * This one "log line" gives us both a metric for alarming and a log entry for debugging, with zero custom SDKs.

### 3. Tracing

* **Service:** **AWS X-Ray**
* **How it Works:**
    * We add the X-Ray SDK to all three microservices.
    * This traces a single user request as it flows from the ALB -> `BANK` service -> `RDS` Database.
    * **Result:** We get a "Service Map" that visually shows bottlenecks. If a bet is slow, X-Ray will tell us *exactly* which service (or database query) is the cause.

### 4. Alerting

* **Service:** **CloudWatch Alarms** + **Amazon SNS**
* **How it Works:**
    1.  **Define Alarms:** We create alarms in CloudWatch that watch our metrics.
    2.  **Send to SNS:** When an alarm triggers, it sends a message to an SNS (Simple Notification Service) topic.
    3.  **Fan Out:** SNS sends that message to all subscribers, such as:
        * **PagerDuty/Opsgenie** (for high-priority, "wake someone up" alerts).
        * **Slack** (for low-priority, "FYI" alerts).
        * **Email**.

### 5. Dashboards & Uptime

* **Service:** **CloudWatch Dashboards** & **CloudWatch Synthetics**
* **How it Works:**
    * **Dashboards:** We build dashboards in CloudWatch that combine everything: metric graphs, Logs Insights queries, and the X-Ray service map. This gives us a "single pane of glass."
    * **Synthetics (Canaries):** For uptime, we don't just wait for something to break. We use Synthetics to run a "Canary" (a small script) every 1 minute that:
        1.  Tries to log in to your site.
        2.  Tries to connect to the `SOCKETS` chat endpoint.
    * If this "Canary" fails, an alarm triggers *immediately*, letting us know the chat is down before users do.


## Addressing Mission-Critical Areas

| Requirement                     | Solution (Using the AWS Native Stack) |
|:--------------------------------| :--- |
| **Bets Processed Correctly**    | **Metric (CloudWatch EMF):** Create an Embedded Metric Format (EMF) log that generates a `bet.failed.count` metric.<br/>**Log (CloudWatch Logs):** The same EMF log provides the full context: `{"_aws": {"Metrics": [{"Name": "bet.failed.count"}]}, "userId": "123", "reason": "insufficient_funds"}`.<br/>**Alert (CloudWatch Alarm):** `ALARM` if `SUM` of `bet.failed.count > 0` in 1 minute. |
| **Rewards Calculated Properly** | **Metric (Standard CloudWatch):** Monitor the standard `AWS/Kafka` metric `SumOffsetLag` for your `REWARDS` consumer group.<br/>**Alert (CloudWatch Alarm):** `ALARM` if `SumOffsetLag > 1000` for 5 consecutive minutes (meaning rewards processing is delayed). |
| **User Abuse / Scraping**       | **Tool (AWS WAF):** Use rate-limiting rules on the ALB to block scrapers and simple brute-force attacks.<br/>**Logs (CloudWatch Logs):** Ship ALB and WAF logs to a CloudWatch Log Group.<br/>**Analysis (Logs Insights):** Create a saved query: `stats count() by clientIp, userAgent \| filter status >= 400 \| sort by count() desc`.<Gbr/>**Alert (Metric Filter):** Create a "Metric Filter" on the log group to count `4xx` errors and create a `waf.blocked.count` metric to alarm on. |
| **Deposits & Withdrawals**      | **Metric (CloudWatch EMF):** Create a `deposit.failed.count` metric using EMF.<br/>**Alert (CloudWatch Alarm):** `ALARM` if `SUM` of `deposit.failed.count > 0` in 1 minute. <br/>**Action:** This alarm triggers an **Amazon SNS** topic, which sends a high-priority notification to **PagerDuty**. |
| **Uptime (e.g., Chat)**         | **Tool (CloudWatch Synthetics):** Create a "Canary" (a Node.js script) that uses a WebSocket client to connect to the `SOCKETS` endpoint.<br/>**How:** The Canary runs on a 1-minute schedule from multiple regions.<br/>**Alert (CloudWatch Alarm):** CloudWatch automatically creates a `CanaryName-SuccessPercent` metric. `ALARM` if `SuccessPercent < 100` for 1 minute. |
| **Casino Admin KPIs**           | **Dashboard (CloudWatch Dashboard):** Create a non-technical dashboard with widgets for:<br/> 1. `SUM` of `deposit.success.value` (from EMF).<br/> 2. `AVERAGE` of `socket.connections.active` (a gauge metric from EMF).<br/> 3. `SUM` of `new.user.registration` (from EMF). |