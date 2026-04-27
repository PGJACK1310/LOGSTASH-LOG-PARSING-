# 🔥 Firewall Log Parsing (Logstash)

Lightweight Logstash pipeline for parsing firewall logs into structured data for Elasticsearch.

Currently supports:
- FortiGate firewall logs

---

## ⚙️ Features

- Syslog input (UDP 514)
- KV-based parsing (fast and scalable)
- Extracts IPs, ports, protocol, action
- Timestamp normalization
- GeoIP enrichment
- Elasticsearch-ready output

---

## 🚀 Usage

1. Clone the repository:
git clone https://github.com/PGJACK1310/LOGSTASH-LOG-PARSING-.git

2. Copy config file to Logstash:
 /etc/logstash/conf.d/

3. Start Logstash:
sudo systemctl start logstash

---

## 📁 Project Structure

.
├── fortigate.conf
└── README.md

---

## 🔮 Roadmap

- Add support for more firewall vendors (Cisco, Palo Alto, etc.)

---

## 👨‍💻 Author

PGJACK1310  
https://github.com/PGJACK1310
