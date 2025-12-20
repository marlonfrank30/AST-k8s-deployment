# F5 Application Study Tool - Kubernetes Deployment

## Overview

This repository contains Kubernetes manifests for deploying the F5 Application Study Tool (AST) stack, which includes OpenTelemetry Collector, Prometheus, and Grafana. The solution collects metrics from F5 BIG-IP devices, stores them in Prometheus, and visualizes them through Grafana dashboards.

The deployment provides comprehensive monitoring and telemetry collection for F5 BIG-IP systems, supporting multiple data types including APM, CGNAT, DNS, DoS, Firewall, GTM, and various security policies.

## Architecture

The deployment consists of three main components:

**OpenTelemetry Collector** - Scrapes metrics from F5 BIG-IP devices using the custom F5 receiver  
**Prometheus** - Time-series database for storing metrics with 1-year retention  
**Grafana** - Visualization and dashboarding platform for analyzing collected metrics

All components are deployed in the `ast` namespace and communicate via Kubernetes services.

---

## Components

### 1. **OpenTelemetry Collector**

Custom collector built with F5-specific receivers to gather telemetry from BIG-IP devices.

#### Key Features:
- **Collection Interval:** 60 seconds
- **Supported Data Types:**
  - F5 APM (Access Policy Manager)
  - F5 CGNAT (Carrier-Grade NAT)
  - F5 DNS
  - F5 DoS (Denial of Service)
  - F5 Firewall
  - F5 GTM (Global Traffic Manager)
  - F5 Policy (API Protection, ASM, Firewall)
  - F5 Profile DoS
- **Exporters:**
  - Local Prometheus (OTLP HTTP)
  - F5 Data Fabric (optional)
  - Debug output for troubleshooting
- **Self-Monitoring:** Exposes Prometheus metrics on port 8888

---

### 2. **Prometheus**

Time-series database configured to receive metrics from the OpenTelemetry Collector via the OTLP write receiver.

#### Configuration:
- **Scrape Intervals:**
  - Prometheus self-monitoring: 1 minute
  - OpenTelemetry Collector: 30 seconds
- **Retention:** 1 year
- **Features:**
  - OTLP write receiver enabled
  - Web lifecycle management enabled
- **Storage:** 100Mi persistent volume

---

### 3. **Grafana**

Visualization platform for creating dashboards and analyzing F5 metrics.

#### Configuration:
- **Default Credentials:**
  - Username: `admin`
  - Password: `admin` (configurable via ConfigMap)
- **Storage:** 100Mi persistent volume for dashboard persistence
- **Port:** 3000

---

## Prerequisites

- Kubernetes cluster (v1.20+)
- kubectl configured with cluster access
- Access to F5 BIG-IP device(s) for monitoring
- Valid credentials for BIG-IP devices

---

## Configuration

### Required ConfigMaps

1. **env-device-secrets** - BIG-IP device credentials
2. **env** - Grafana and sensor credentials
3. **bigip-scraper-config** - OpenTelemetry Collector configuration
4. **prometheus-cm0** - Prometheus scrape configuration

### BIG-IP Device Configuration

Update the `env-device-secrets` ConfigMap with your BIG-IP credentials:

```yaml
data:
  BIGIP_PASSWORD_1: "YourSecurePassword"
```

Update the BIG-IP endpoint in `bigip-scraper-config`:

```yaml
receivers.yaml: |
  bigip/1:
    endpoint: https://YOUR_BIGIP_IP
    username: admin
    password: ${env:BIGIP_PASSWORD_1}
```

### F5 Data Fabric (Optional)

If exporting to F5 Data Fabric, update the sensor credentials in the `env` ConfigMap:

```yaml
data:
  SENSOR_ID: YOUR_SENSOR_ID
  SENSOR_SECRET_TOKEN: YOUR_SENSOR_TOKEN
```

---

## Deployment Instructions

### 1. **Deploy All Resources**

Apply the combined manifest to deploy all components:

```bash
kubectl apply -f config/application-study-tool.yaml
```

This will create:
- Namespace: `ast`
- ConfigMaps (4)
- PersistentVolumeClaims (2)
- Deployments (3)
- Services (3)

### 2. **Verify Deployment**

Check that all pods are running:

```bash
kubectl get pods -n ast
```

Expected output:
```
NAME                              READY   STATUS    RESTARTS   AGE
otel-collector-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
prometheus-xxxxxxxxxx-xxxxx       1/1     Running   0          2m
grafana-xxxxxxxxxx-xxxxx          1/1     Running   0          2m
```

### 3. **Verify Services**

```bash
kubectl get svc -n ast
```

Expected output:
```
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
otel-collector   ClusterIP   10.x.x.x        <none>        8888/TCP   2m
prometheus       ClusterIP   10.x.x.x        <none>        9090/TCP   2m
grafana          ClusterIP   10.x.x.x        <none>        3000/TCP   2m
```

---

## Accessing the Services

### Port Forwarding (Development)

**Grafana:**
```bash
kubectl port-forward -n ast svc/grafana 3000:3000
```
Access at: http://localhost:3000

**Prometheus:**
```bash
kubectl port-forward -n ast svc/prometheus 9090:9090
```
Access at: http://localhost:9090

**OpenTelemetry Collector Metrics:**
```bash
kubectl port-forward -n ast svc/otel-collector 8888:8888
```
Access at: http://localhost:8888/metrics

### Production Ingress (Recommended)

Create Ingress resources to expose services externally:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ast-ingress
  namespace: ast
spec:
  rules:
  - host: grafana.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grafana
            port:
              number: 3000
```

---

## Configuration Deep Dive

### OpenTelemetry Collector Pipeline

The collector uses a modular pipeline architecture:

```
receivers (BIG-IP) → processors (batch) → exporters (Prometheus, Debug)
```

**Receivers:**
- `bigip/1` - Connects to BIG-IP device and collects metrics

**Processors:**
- `batch/local` - Batches metrics for efficient processing
- `batch/f5-datafabric` - Batches for F5 Data Fabric export (if enabled)

**Exporters:**
- `otlphttp/metrics-local` - Sends to local Prometheus
- `debug/bigip` - Outputs debug information
- `otlp/f5-datafabric` - Sends to F5 Data Fabric (optional)

### Data Types Collected

Configure which data types to collect by modifying the `receivers.yaml` section:

```yaml
bigip/1:
  data_types:
    f5.apm:
      enabled: true    # Access Policy Manager
    f5.dos:
      enabled: true    # DoS protection
    f5.policy.asm:
      enabled: true    # Application Security Manager
    # ... additional types
```

---

## Monitoring and Troubleshooting

### View Collector Logs

```bash
kubectl logs -n ast deployment/otel-collector -f
```

### View Prometheus Logs

```bash
kubectl logs -n ast deployment/prometheus -f
```

### View Grafana Logs

```bash
kubectl logs -n ast deployment/grafana -f
```

### Check Metrics Collection

Verify metrics are being scraped:

```bash
# Check OpenTelemetry Collector metrics
kubectl exec -n ast deployment/otel-collector -- wget -qO- localhost:8888/metrics

# Check Prometheus targets
kubectl port-forward -n ast svc/prometheus 9090:9090
# Open browser: http://localhost:9090/targets
```

### Debug OpenTelemetry Collector

The collector includes a debug exporter. Check logs for output:

```bash
kubectl logs -n ast deployment/otel-collector | grep -A 10 "ResourceMetrics"
```

### Common Issues

**Collector can't reach BIG-IP:**
- Verify network connectivity from cluster to BIG-IP
- Check firewall rules allow HTTPS (443) from cluster
- Verify BIG-IP credentials in ConfigMap

**No metrics in Prometheus:**
- Check collector logs for export errors
- Verify Prometheus OTLP endpoint is accessible
- Check Prometheus configuration and targets

**Grafana can't connect to Prometheus:**
- Add Prometheus as a data source in Grafana
- Use service DNS: `http://prometheus.ast.svc.cluster.local:9090`

---

## Security Considerations

### Credentials Management

**Current Implementation:** Credentials are stored in ConfigMaps (not recommended for production)

**Recommended:** Use Kubernetes Secrets with encryption at rest:

```bash
kubectl create secret generic bigip-credentials \
  --from-literal=password='YourSecurePassword' \
  -n ast
```

Update deployment to reference secrets:

```yaml
env:
- name: BIGIP_PASSWORD_1
  valueFrom:
    secretKeyRef:
      name: bigip-credentials
      key: password
```

### Network Policies

Implement network policies to restrict traffic:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ast-netpol
  namespace: ast
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ast
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: ast
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 53
    - protocol: UDP
      port: 53
```

### TLS Configuration

The BIG-IP receiver currently uses `insecure_skip_verify: true`. For production:

1. Obtain the BIG-IP CA certificate
2. Create a ConfigMap with the certificate
3. Update the receiver configuration:

```yaml
tls:
  ca_file: /etc/ssl/certs/bigip-ca.crt
  insecure_skip_verify: false
```

---

## Scaling and Performance

### Resource Requests/Limits

Add resource constraints to prevent overconsumption:

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

### Prometheus Storage

Adjust retention and storage based on requirements:

```yaml
args:
- --storage.tsdb.retention.time=1y
- --storage.tsdb.retention.size=50GB
```

Update PVC size:

```yaml
resources:
  requests:
    storage: 100Gi  # Increase as needed
```

---

## Backup and Disaster Recovery

### Backup Grafana Dashboards

```bash
kubectl exec -n ast deployment/grafana -- tar czf - /var/lib/grafana | \
  cat > grafana-backup-$(date +%Y%m%d).tar.gz
```

### Backup Prometheus Data

```bash
kubectl exec -n ast deployment/prometheus -- tar czf - /prometheus | \
  cat > prometheus-backup-$(date +%Y%m%d).tar.gz
```

### Restore from Backup

```bash
cat grafana-backup-YYYYMMDD.tar.gz | \
  kubectl exec -i -n ast deployment/grafana -- tar xzf - -C /
```

---

## Maintenance

### Update BIG-IP Credentials

```bash
kubectl edit configmap env-device-secrets -n ast
# Edit the password
kubectl rollout restart deployment/otel-collector -n ast
```

### Update Collector Configuration

```bash
kubectl edit configmap bigip-scraper-config -n ast
# Make changes
kubectl rollout restart deployment/otel-collector -n ast
```

### Add Additional BIG-IP Devices

Edit the `bigip-scraper-config` ConfigMap and add new receivers:

```yaml
receivers.yaml: |
  bigip/1:
    endpoint: https://192.168.0.249
    # ... configuration
  bigip/2:  # New device
    endpoint: https://192.168.0.250
    username: admin
    password: ${env:BIGIP_PASSWORD_2}
```

Update pipelines to include the new receiver:

```yaml
pipelines.yaml: |
  metrics/local:
    receivers:
    - bigip/1
    - bigip/2  # Add new receiver
```

import json files into grafana for pre-populated dashboards as examples
```bash
cd import
```

```json
GTM-1766186184691.json
iRules-1766186209607.json
Overview-1766186317519.json
Pools-1766186357208.json
Virtual Server-1766186388431.json
Device WAF Overview-1766186405629.json
Device WAF Overview-1766186442866.json
Top N-1766186423918.json
```
---

## Clean Up

### Remove All Resources

```bash
kubectl delete -f application-study-tool.yaml
```

### Remove Namespace

```bash
kubectl delete namespace ast
```

**Warning:** This will delete all persistent data including Prometheus metrics and Grafana dashboards.

---

## Version Information

- **OpenTelemetry Collector:** Custom F5 build (ghcr.io/f5devcentral/application-study-tool/otel_custom_collector)
- **Prometheus:** prom/prometheus (SHA: 7a34573f0b9c952286b33d537f233cd5b708e12263733aa646e50c33f598f16c)
- **Grafana:** grafana/grafana (SHA: 6128afd8174f01e39a78341cb457588f723bbb9c3b25c4d43c4b775881767069)
- **Kubernetes:** Tested on v1.20+

---

## Contributing

When contributing configurations:

1. Test in a non-production environment
2. Document any new features or changes
3. Update this README with deployment instructions
4. Verify all components are working correctly

---

## License

This project is intended for operational automation within F5 environments.  
Use at your own risk and validate in a test environment prior to production deployment.

---

## 🧾 References & Resources

- [F5 Application Study Tool](https://github.com/f5devcentral/application-study-tool)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [F5 BIG-IP API Reference](https://clouddocs.f5.com/)

---

## ⚙️ Notes

- Ensure your Kubernetes cluster has sufficient resources for all components
- Default storage class must support ReadWriteOnce access mode
- BIG-IP devices must be accessible from the Kubernetes cluster network
- Consider using NodePort or LoadBalancer services for external access in production
- Review and adjust retention policies based on storage capacity
- Recommended to test changes in a non-production setup before deployment.


---

## 🧑‍💻 Support

For issues related to:
- **F5 Components:** Refer to F5 support channels
- **Kubernetes Deployment:** Check Kubernetes logs and events
- **OpenTelemetry:** Review collector logs and configuration
- **Prometheus/Grafana:** Consult respective project documentation


## 🧑‍💻 Author
**Marlon Frank**  
*Network and Application Security & F5 Automation Engineer*  
