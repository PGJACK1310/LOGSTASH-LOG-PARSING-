# Logstash Pipeline — FortiGate Firewall Log Ingestion

A production-ready Logstash pipeline that ingests **FortiGate syslog traffic** over UDP, parses all key-value fields, enriches them with GeoIP data, and ships structured logs to Elasticsearch.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Configuration](#configuration)
- [Pipeline Breakdown](#pipeline-breakdown)
  - [Input](#input)
  - [Filter — Block 1: FortiGate Standard Logs](#filter--block-1-fortigate-standard-logs)
  - [Filter — Vendor Identification](#filter--vendor-identification)
  - [Output](#output)
- [Field Reference](#field-reference)
- [Log Categories](#log-categories)
- [Elasticsearch Index Strategy](#elasticsearch-index-strategy)
- [Customization](#customization)
- [Troubleshooting](#troubleshooting)

---

## Overview

This pipeline is designed to handle the full range of FortiGate log subtypes in a single, maintainable configuration:

| Log Type | Subtype Examples |
|---|---|
| `traffic` | Forward, local, sniffer |
| `event` | VPN, system, user (authentication) |
| `utm` | Web filter, IPS, application control |
| `ddos` | DDoS protection events |
| `nat` | NAT translation events |
| `snmp` | SNMP trap events |

The key design decision is the use of a **single `kv` (key-value) filter** to replace 50+ individual grok patterns — making the pipeline dramatically simpler, faster, and easier to extend.

---

## Architecture

```
FortiGate Device
      │
      │  UDP Syslog (Port 514)
      ▼
┌─────────────┐
│   Logstash  │
│  ─────────  │
│   INPUT     │  ← UDP listener on port 514
│   FILTER    │  ← Header parse → KV parse → Field promotion
│             │  ← Timestamp normalization → GeoIP enrichment
│   OUTPUT    │  ← Route to Elasticsearch index
└─────────────┘
      │
      ▼
Elasticsearch
  ├── fortigate-YYYY.MM.dd        (normal logs)
  └── grokparsefailure-YYYY.MM.dd (failed parses)
```

---

## Prerequisites

| Component | Version | Notes |
|---|---|---|
| Logstash | 7.x / 8.x | Tested on both |
| Elasticsearch | 7.x / 8.x | SSL enabled |
| GeoIP Database | MaxMind GeoLite2 | Bundled with Logstash |
| FortiGate Firmware | Any | Logs must be in standard `key=value` syslog format |

---

## Configuration

### 1. FortiGate Syslog Setup

Configure your FortiGate device to send syslog to the Logstash host:

```
config log syslogd setting
    set status enable
    set server <LOGSTASH_HOST_IP>
    set port 514
    set mode udp
    set facility local7
    set format default
end
```

### 2. Update Elasticsearch Credentials

Open the pipeline file and replace the placeholder values in the **output** section:

```ruby
hosts    => ["https://<YOUR_ES_HOST>:9200"]
user     => "elastic"
password => "<YOUR_PASSWORD>"
```

### 3. Set the Correct Timezone

In **Step 4** of the filter, update the timezone to match your FortiGate's configured timezone:

```ruby
date {
  match    => [ "fg_timestamp", "yyyy-MM-dd HH:mm:ss" ]
  target   => "@timestamp"
  timezone => "Asia/Kolkata"   # ← Change this
}
```

Common values: `UTC`, `Asia/Kolkata`, `America/New_York`, `Europe/London`

### 4. Deploy the Pipeline

Place the configuration file in your Logstash pipelines directory and restart:

```bash
cp fortigate.conf /etc/logstash/conf.d/
systemctl restart logstash
```

---

## Pipeline Breakdown

### Input

```ruby
udp {
  port => 514
  type => "syslog"
}
```

Listens for raw UDP syslog messages on port **514**. All incoming events are tagged with `type = "syslog"`.

> **Note:** Ensure port 514/UDP is open on your firewall and that Logstash has permission to bind to it (may require `sudo` or `authbind` on Linux).

---

### Filter — Block 1: FortiGate Standard Logs

Detection trigger: message contains `devname=` (a reliable marker present in all standard FortiGate log lines).

The filter executes in **8 sequential steps**:

#### Step 1 — Header Parse (Grok)

Strips the syslog priority prefix and extracts the date/time header:

```
<%PRIORITY%>date=YYYY-MM-DD time=HH:MM:SS <rest of message>
```

On failure, the tag `_fg_header_fail` is applied and subsequent steps are skipped.

#### Step 2 — Key-Value Parsing

A single `kv` filter parses **all** `key=value` pairs from the log body in one pass. Handles:
- Quoted values: `msg="tunnel is down"`
- Unquoted values: `proto=6`
- All 50+ FortiGate field names across every log subtype

#### Step 3 — Field Promotion

Parsed fields are renamed from the nested `[fg][field]` structure to clean top-level field names (e.g., `[fg][srcip]` → `src_ip`). A special fallback handles NAT logs that use `saddr` instead of `srcip`.

#### Step 4 — Timestamp Normalization

Combines `fg_date` and `fg_time` into a single `fg_timestamp` field, then parses it into the Elasticsearch `@timestamp` field using the configured timezone.

#### Step 5 — Type Conversion

Numeric fields are cast from strings to integers for correct sorting and range queries in Kibana:

`src_port`, `dst_port`, `proto`, `duration`, `sent_bytes`, `recv_bytes`, `sent_pkt`, `recv_pkt`, `cpu`, `mem`, `port_begin`, `port_end`, `remote_port`, `local_port`

#### Step 6 — GeoIP Enrichment

Adds geographic location data (country, city, coordinates) for both source and destination IPs:

```
src_geoip.country_name, src_geoip.location (geo_point)
dst_geoip.country_name, dst_geoip.location (geo_point)
```

#### Step 7 — Log Category Tagging

Sets a `log_category` field for easy Kibana dashboard filtering:

| Condition | `log_category` |
|---|---|
| `fg_type = traffic` | `network_traffic` |
| `fg_type = utm` | `security` |
| `fg_type = event`, `fg_subtype = vpn` | `vpn` |
| `fg_type = event`, `fg_subtype = system` | `system` |
| `fg_type = event`, `fg_subtype = user` | `authentication` |

#### Step 8 — Cleanup

Removes all temporary processing fields (`fg_body`, `fg_date`, `fg_time`, `fg_timestamp`, `fg`) to keep the final document lean.

---

### Filter — Vendor Identification

For events that already have `fg_type` or `fg_subtype` set (e.g., pre-enriched events), vendor metadata is added:

```ruby
vendor      => "fortigate"
device_type => "firewall"
```

---

### Output

Events are routed to Elasticsearch based on parse status:

| Condition | Index |
|---|---|
| Parse failed (`_grokparsefailure` tag present) | `grokparsefailure-YYYY.MM.dd` |
| Successfully parsed FortiGate log | `fortigate-YYYY.MM.dd` |

Both outputs use SSL with verification mode set to `none` (suitable for self-signed certificates).

---

## Field Reference

| Top-Level Field | Source FortiGate Key | Description |
|---|---|---|
| `devname` | `devname` | Device hostname |
| `devid` | `devid` | Device serial number |
| `fg_type` | `type` | Log type (traffic, event, utm) |
| `fg_subtype` | `subtype` | Log subtype |
| `level` | `level` | Severity level |
| `event_name` | `logdesc` | Human-readable log description |
| `event_message` | `msg` | Detailed log message |
| `src_ip` | `srcip` / `saddr` | Source IP address |
| `dst_ip` | `dstip` | Destination IP address |
| `src_port` | `srcport` | Source port |
| `dst_port` | `dstport` | Destination port |
| `srcintf` | `srcintf` | Source interface name |
| `dstintf` | `dstintf` | Destination interface name |
| `src_country` | `srccountry` | Source country |
| `dst_country` | `dstcountry` | Destination country |
| `user` | `user` | Authenticated username |
| `action` | `action` | Policy action taken |
| `status` | `status` | Event status |
| `proto` | `proto` | IP protocol number |
| `duration` | `duration` | Session duration (seconds) |
| `sent_bytes` | `sentbyte` | Bytes sent |
| `recv_bytes` | `rcvdbyte` | Bytes received |
| `sent_pkt` | `sentpkt` | Packets sent |
| `recv_pkt` | `rcvdpkt` | Packets received |
| `vpn_tunnel` | `vpntunnel` | VPN tunnel name |
| `assign_ip` | `assignip` | IP assigned to VPN client |
| `remote_ip` | `remip` | VPN remote IP |
| `local_ip` | `locip` | VPN local IP |
| `nat_ip` | `nat` | NAT translated IP |
| `tunnel_ip` | `tunnelip` | IPSec tunnel IP |
| `community` | `community` | SNMP community string |
| `snmp_version` | `version` | SNMP version |
| `cpu` | `cpu` | CPU usage % |
| `mem` | `mem` | Memory usage % |
| `role` | `role` | Interface or tunnel role |
| `result` | `result` | Operation result |
| `group` | `group` | User group |

---

## Log Categories

Use the `log_category` field in Kibana to build category-specific dashboards without needing complex filter queries:

```
log_category : "network_traffic"   → Firewall traffic logs
log_category : "security"          → UTM/threat events
log_category : "vpn"               → VPN tunnel events
log_category : "system"            → System events
log_category : "authentication"    → User login/logout events
```

---

## Elasticsearch Index Strategy

Logs are written to daily rolling indices (`fortigate-YYYY.MM.dd`). It is recommended to apply an **Index Lifecycle Management (ILM)** policy to manage retention:

```json
PUT _ilm/policy/fortigate-policy
{
  "policy": {
    "phases": {
      "hot":    { "actions": { "rollover": { "max_age": "1d" } } },
      "warm":   { "min_age": "7d",  "actions": { "shrink": { "number_of_shards": 1 } } },
      "delete": { "min_age": "90d", "actions": { "delete": {} } }
    }
  }
}
```

---

## Customization

**Add a new FortiGate field:** Add the key name to the `include_keys` list in the `kv` block, then add a `rename` entry in the `mutate` block under Step 3.

**Change the listening port:** Update `port => 514` in the input section and adjust your FortiGate syslog configuration to match.

**Add a new log category:** Extend the `if/else if` chain in Step 7 with the new `fg_type`/`fg_subtype` combination.

**Enable SSL verification:** Change `ssl_verification_mode => "none"` to `"full"` and provide the CA certificate path via `ssl_certificate_authorities`.

---

## Troubleshooting

**No events arriving in Elasticsearch**
- Verify FortiGate is sending to the correct Logstash IP and port 514/UDP.
- Check that no firewall is blocking UDP 514 between FortiGate and Logstash.
- Run `tcpdump -i any udp port 514` on the Logstash host to confirm packets are arriving.

**Events landing in `grokparsefailure` index**
- The log line does not start with the expected `<%INT%>date=` header format.
- Check the raw message in Kibana's `grokparsefailure-*` index to identify the format difference.

**`_fg_header_fail` tag present**
- The syslog priority or date/time header format differs from the expected pattern.
- Verify the FortiGate log format is set to `default` (not `csv` or `cef`).

**Timestamp incorrect**
- Ensure the `timezone` value in Step 4 matches the timezone configured on your FortiGate device.

**GeoIP not enriching private IPs**
- This is expected behavior. MaxMind GeoIP databases do not contain entries for RFC1918 private IP ranges.
