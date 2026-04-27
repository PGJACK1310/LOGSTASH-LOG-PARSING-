# 🔥 FortiGate Firewall Log Parsing (Logstash)

This project provides a high-performance Logstash pipeline for parsing FortiGate firewall logs using a hybrid **Grok + KV parsing strategy**.

It is optimized to replace complex multi-pattern grok configurations with a single scalable pipeline.

---

## 📌 Overview

This Logstash configuration ingests FortiGate syslog data and transforms it into structured, enriched events ready for Elasticsearch and Kibana.

The parser:
- Detects FortiGate logs using `devname=` marker
- Extracts timestamp and metadata
- Parses all key-value pairs dynamically
- Normalizes field names
- Enriches logs with GeoIP data
- Categorizes logs for analytics

---

## ⚙️ Pipeline Architecture

### 1. Input
- Listens for syslog messages over UDP

```logstash
input {
  udp {
    port => 514
    type => "syslog"
  }
}
