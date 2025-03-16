# Pi-hole Monitoring Dashboard: Implementation on Raspberry Pi 5 with MicroK8s

> **Note:** For security and privacy reasons, all IP addresses, MAC addresses, and hostnames in this documentation have been changed from their actual values. The network configuration principles remain accurate, but the specific addresses shown are examples only.

## Project Overview

This repository documents my implementation of a real-time monitoring dashboard for Pi-hole on a Raspberry Pi 5 running Ubuntu 24.04 LTS, using MicroK8s, Pi-hole Exporter, Prometheus, and Grafana. The dashboard visualizes Pi-hole DNS metrics such as blocked domains, query rates, and top sources, enhancing network management and troubleshooting capabilities.


## System Specifications

![Raspberry Pi 5 Hardware](/images/kubepi5physical.JPG)
*My Raspberry Pi 5 setup*

- **Hardware**: Raspberry Pi 5 (4GB RAM)
- **Storage**: 64GB microSD card
- **Operating System**: Ubuntu 24.04 LTS (64-bit)
- **Network**: Static IP configuration on Ethernet (eth0), example IP: 192.178.1.5
- **Router**: AT&T Fiber (example gateway: 192.178.1.1)
- **Existing Setup**: Pi-hole running as a pod in MicroK8s, accessible at http://192.178.1.5/admin/

## Installation Steps

### 1. Verify MicroK8s Environment

```yaml
snap list | grep microk8s          # Confirmed v1.32.2
microk8s status                    # Verified running with DNS and Helm
microk8s kubectl get nodes         # Showed bryan-pi as Ready
```

### 2. Confirm Pi-hole Pod
- Checked running pods:

```yaml
microk8s kubectl get pods          # pihole-645b4448c7-qmvw7 (1/1 Running, 44h)
```
- Verified services: Pi-hole accessible at 192.178.1.5:80.

### 3. Install Pi-hole Exporter
- Deployed with `pihole-exporter.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pihole-exporter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pihole-exporter
  template:
    metadata:
      labels:
        app: pihole-exporter
    spec:
      containers:
      - name: pihole-exporter
        image: ekofr/pihole-exporter:latest
        env:
        - name: PIHOLE_HOSTNAME
          value: "192.178.1.5"
        - name: PIHOLE_PASSWORD
          value: "<your-pihole-web-password>"
        ports:
        - containerPort: 9617
        resources:
          limits:
            cpu: "100m"
            memory: "128Mi"
```

- Applied: `microk8s kubectl apply -f pihole-exporter.yaml.`

### 4. Install Prometheus

- Deployed with `prometheus.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:v2.51.0
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: config-volume
          mountPath: /etc/prometheus
      volumes:
      - name: config-volume
        configMap:
          name: prometheus-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
      - job_name: 'pihole'
        static_configs:
          - targets: ['pihole-exporter:9617']
```
- Applied: `microk8s kubectl apply -f prometheus.yaml`.

### 5. Install Grafana

Deployed with `grafana.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:latest
        ports:
        - containerPort: 4000
        resources:
          limits:
            cpu: "200m"
            memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
spec:
  selector:
    app: grafana
  ports:
  - port: 4000
    targetPort: 4000
  type: NodePort
```
- Applied: `microk8s kubectl apply -f grafana.yaml`.

### 6. Post-Installation Configuration

- Added Prometheus as Grafana data source: `http://prometheus:9090`.
- Imported Grafana dashboard ID `10176` (Pi-hole Exporter).
- Accessed Grafana at `http://192.178.1.5:31535 (NodePort)`.

![Grafana Dashboard](/images/Screenshot%202025-03-16%20181927.png)
Sample Grafana dashboard showing Pi-hole metrics

## Implementation Steps

### 1. Verify MicroK8s Environment
- Confirmed MicroK8s installation: `snap list | grep microk8s` showed version v1.32.2.
- Checked cluster status: `microk8s status` confirmed running with DNS and Helm addons enabled.
- Verified node: `microk8s kubectl get nodes` showed `bryan-pi` as Ready.

### 2. Confirm Pi-hole Pod
- Identified existing Pi-hole pod: `pihole-645b4448c7-qmvw7` (1/1 Running, 44h).
- Checked services: `microk8s kubectl get svc` showed only `kubernetes` service, assumed Pi-hole at `192.178.1.5:80`.

# Troubleshooting Challenges & Solutions

## 1. Pi-hole Exporter Metrics Failure

Challenge: Initial `http://192.178.1.5:9617/metrics` failed with "connection refused"; later showed only Go metrics.
Solution:

- Adjusted `PIHOLE_HOSTNAME` to `192.178.1.5` (removed port and path).
- Corrected `PIHOLE_PASSWORD` after verifying logs (`/api/auth 404`).
- Validated metrics with `curl http://192.178.1.5:9617/metrics`.

## 2. Grafana Access Denied

Challenge: `http://192.178.1.5:4000` failed with "connection refused".
Solution:

- Tested locally: `curl http://127.0.0.1:4000` succeeded.
- Used NodePort (`31535`) instead of port-forwarding for stable access.

## Performance Verification
- Verified Prometheus:

```yaml
curl http://127.0.0.1:9090/api/v1/query?query=pihole_domains_being_blocked`
#Returned: `131371`
```

Dashboard metrics:
- Domains blocked: 131,371
- Total queries: ~56,000
- Ads blocked: ~11,000

Response time: Queries processed in <10ms.

## Key Achievements

- Deployed a real-time Pi-hole monitoring dashboard with MicroK8s.
- Integrated Prometheus and Grafana for comprehensive DNS metrics.
- Resolved networking and API configuration challenges.
- Achieved accessible visualization at `http://192.178.1.5:31535`.

## Skills Demonstrated

- Kubernetes administration (MicroK8s pod and service management)
- Monitoring setup (Prometheus scraping, Grafana visualization)
- Network troubleshooting (NodePort, DNS resolution)
- Technical documentation and YAML configuration

## Future Improvements

- Automate backups of MicroK8s and Pi-hole configurations.
- Optimize resource usage with lighter container images.
- Add alerting rules in Prometheus for query spikes or failures.
- Enhance dashboard with custom panels for detailed client analysis.

## Resources

- [Pi-hole Exporter](https://github.com/eko/pihole-exporter)
- [Prometheus Documentation](https://prometheus.io/docs/introduction/overview/)
- [Grafana Documentation](https://grafana.com/docs/)
- [MicroK8s Documentation](https://microk8s.io/docs)
