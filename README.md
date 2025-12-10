# Monitoring Tools Setup Guide

This project provides a set of monitoring tools used for the **INT531 Site Reliability Engineering** course.

---

## 🚀 Installation Guide

### 1. Install Node Exporter

Download the latest Node Exporter release from the official Prometheus website:

🔗 https://prometheus.io/download/#node_exporter

#### 1.1. Download the Linux release

(_Example screenshot: `./docs/node_exporter_download.png`_)

#### 1.2. Run the following commands:

> ⚠️ **Important:** Replace `<VERSION>`, `<OS>`, and `<ARCH>` with the correct values from the download page.

```bash
# Download the Node Exporter tar.gz
wget https://github.com/prometheus/node_exporter/releases/download/v<VERSION>/node_exporter-<VERSION>.<OS>-<ARCH>.tar.gz

# Extract the archive
tar xvfz node_exporter-*.*-amd64.tar.gz

# Move into the extracted folder
cd node_exporter-*.*-amd64

# Start Node Exporter
./node_exporter
```

### 📦 2. Start the Monitoring Stack

Once Node Exporter is installed, start the full monitoring stack using Docker Compose:

```
docker compose -f monitoring.compose.yml up -d
```

This will launch Prometheus, Grafana, and the other configured services in detached mode.
