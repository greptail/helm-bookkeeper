# Apache BookKeeper Helm Chart

A Helm chart for deploying Apache BookKeeper StatefulSet cluster on Kubernetes with automatic metadata formatting and persistent storage.

## Prerequisites

- Kubernetes 1.19+
- Helm 3.2.0+
- Apache ZooKeeper cluster running and accessible
- Persistent Volume provisioner support in the underlying infrastructure

## Installing the Chart

### Quick Start

```bash
# Add your ZooKeeper service URI and install
helm install my-bookkeeper ./bookkeeper \
  --set zookeeper.metadataServiceUri="zk://zookeeper:2181/ledgers"
```

### Install with Custom Values

```bash
# Create custom values file
cat > my-values.yaml << EOF
replicaCount: 5
zookeeper:
  metadataServiceUri: "zk://zookeeper-service:2181/bookkeeper"
persistence:
  size: 20Gi
  storageClass: "fast-ssd"
resources:
  requests:
    memory: 2Gi
    cpu: 1000m
  limits:
    memory: 4Gi
    cpu: 2000m
EOF

# Install with custom values
helm install my-bookkeeper ./bookkeeper -f my-values.yaml
```

### Install in Specific Namespace

```bash
# Create namespace
kubectl create namespace bookkeeper

# Install in namespace
helm install my-bookkeeper ./bookkeeper \
  --namespace bookkeeper \
  --set zookeeper.metadataServiceUri="zk://zookeeper:2181/ledgers"
```

## Configuration

### Required Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `zookeeper.metadataServiceUri` | ZooKeeper metadata service URI | `zk://zookeeper:2181/ledgers` |

### Common Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of BookKeeper nodes | `3` |
| `image.repository` | BookKeeper image repository | `apache/bookkeeper` |
| `image.tag` | BookKeeper image tag | `4.16.1` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |

### BookKeeper Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `bookkeeper.config.bookiePort` | Bookie service port | `3181` |
| `bookkeeper.config.httpServerPort` | HTTP server port | `8080` |
| `bookkeeper.journalDirectory` | Journal data directory | `/opt/bookkeeper/data/journal` |
| `bookkeeper.ledgerDirectories` | Ledger data directory | `/opt/bookkeeper/data/ledgers` |
| `bookkeeper.indexDirectories` | Index data directory | `/opt/bookkeeper/data/index` |

### Metadata Formatting

| Parameter | Description | Default |
|-----------|-------------|---------|
| `bookkeeper.metaformat.enabled` | Enable automatic metadata formatting | `true` |
| `bookkeeper.metaformat.force` | Force format (development only) | `false` |

### Storage Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `persistence.enabled` | Enable persistent storage | `true` |
| `persistence.size` | Storage size | `10Gi` |
| `persistence.storageClass` | Storage class | `""` |
| `persistence.accessMode` | Access mode | `ReadWriteOnce` |

### Resource Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `resources.requests.cpu` | CPU request | `500m` |
| `resources.requests.memory` | Memory request | `1Gi` |
| `resources.limits.cpu` | CPU limit | `1000m` |
| `resources.limits.memory` | Memory limit | `2Gi` |

### Service Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `service.type` | Service type | `ClusterIP` |
| `service.port` | Bookie service port | `3181` |
| `service.httpPort` | HTTP service port | `8080` |
| `headlessService.enabled` | Enable headless service | `true` |

### Monitoring Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `serviceMonitor.enabled` | Enable Prometheus ServiceMonitor | `false` |
| `serviceMonitor.interval` | Scrape interval | `30s` |
| `serviceMonitor.scrapeTimeout` | Scrape timeout | `10s` |

## Usage Examples

### Development Environment

```bash
# Small cluster for development
helm install bookkeeper-dev ./bookkeeper \
  --set replicaCount=1 \
  --set zookeeper.metadataServiceUri="zk://zookeeper:2181/ledgers" \
  --set persistence.size=5Gi \
  --set resources.requests.memory=512Mi \
  --set resources.requests.cpu=250m
```

### Production Environment

```bash
# Production cluster with monitoring
helm install bookkeeper-prod ./bookkeeper \
  --set replicaCount=5 \
  --set zookeeper.metadataServiceUri="zk://zookeeper-prod:2181/bookkeeper" \
  --set persistence.size=100Gi \
  --set persistence.storageClass="fast-ssd" \
  --set resources.requests.memory=4Gi \
  --set resources.requests.cpu=2000m \
  --set resources.limits.memory=8Gi \
  --set resources.limits.cpu=4000m \
  --set serviceMonitor.enabled=true
```

### High Availability Setup

```bash
# HA cluster with anti-affinity
cat > ha-values.yaml << EOF
replicaCount: 7
zookeeper:
  metadataServiceUri: "zk://zk-1:2181,zk-2:2181,zk-3:2181/bookkeeper"

persistence:
  size: 50Gi
  storageClass: "ssd-retain"

resources:
  requests:
    memory: 3Gi
    cpu: 1500m
  limits:
    memory: 6Gi
    cpu: 3000m

affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app.kubernetes.io/name
          operator: In
          values:
          - bookkeeper
      topologyKey: kubernetes.io/hostname

nodeSelector:
  node-type: "bookkeeper"

tolerations:
- key: "bookkeeper"
  operator: "Equal"
  value: "true"
  effect: "NoSchedule"
EOF

helm install bookkeeper-ha ./bookkeeper -f ha-values.yaml
```

## Upgrading

### Upgrade BookKeeper Version

```bash
# Update to new version
helm upgrade my-bookkeeper ./bookkeeper \
  --set image.tag="4.17.0" \
  --reuse-values
```

### Scale the Cluster

```bash
# Scale to 7 nodes
helm upgrade my-bookkeeper ./bookkeeper \
  --set replicaCount=7 \
  --reuse-values
```

### Update Configuration

```bash
# Update storage size
helm upgrade my-bookkeeper ./bookkeeper \
  --set persistence.size=50Gi \
  --reuse-values
```

## Troubleshooting

### Check Pod Status

```bash
# Check pod status
kubectl get pods -l app.kubernetes.io/name=bookkeeper

# Check logs
kubectl logs -l app.kubernetes.io/name=bookkeeper -f

# Check specific pod
kubectl logs bookkeeper-0 -c bookkeeper
```

### Check Metadata Format Job

```bash
# Check metadata format job
kubectl get jobs -l job=metaformat

# Check job logs
kubectl logs job/my-bookkeeper-metaformat
```

### Check Storage

```bash
# Check persistent volumes
kubectl get pv,pvc -l app.kubernetes.io/name=bookkeeper

# Check storage usage
kubectl exec bookkeeper-0 -- df -h /opt/bookkeeper/data
```

### Check BookKeeper Health

```bash
# Check bookie health via HTTP
kubectl port-forward bookkeeper-0 8080:8080
curl http://localhost:8080/heartbeat

# Check bookie state
curl http://localhost:8080/api/v1/bookie/state
```

### Common Issues

#### Metadata Format Issues

```bash
# Manual metadata format (if needed)
kubectl exec bookkeeper-0 -- /opt/bookkeeper/bin/bookkeeper shell metaformat

# Check metadata exists
kubectl exec bookkeeper-0 -- /opt/bookkeeper/bin/bookkeeper shell metaformat -n
```

#### ZooKeeper Connection Issues

```bash
# Test ZooKeeper connectivity
kubectl exec bookkeeper-0 -- /opt/bookkeeper/bin/bookkeeper shell listbookies

# Check ZooKeeper path
kubectl exec bookkeeper-0 -- /opt/bookkeeper/bin/bookkeeper shell ls /
```

#### Storage Issues

```bash
# Check disk usage
kubectl exec bookkeeper-0 -- du -sh /opt/bookkeeper/data/*

# Check permissions
kubectl exec bookkeeper-0 -- ls -la /opt/bookkeeper/data/
```

## Monitoring

### Enable Prometheus Monitoring

```bash
# Install with monitoring enabled
helm upgrade my-bookkeeper ./bookkeeper \
  --set serviceMonitor.enabled=true \
  --reuse-values
```

### Key Metrics to Monitor

- **Bookie Health**: `/heartbeat` endpoint
- **Ledger Count**: Number of ledgers per bookie
- **Disk Usage**: Journal and ledger disk usage
- **Memory Usage**: JVM heap and direct memory
- **Network I/O**: Read/write operations per second

### Grafana Dashboard

Access BookKeeper metrics at:
- Bookie state: `http://<service>:8080/api/v1/bookie/state`
- Metrics: `http://<service>:8080/metrics`

## Uninstalling

### Remove the Chart

```bash
# Uninstall the release
helm uninstall my-bookkeeper

# Remove persistent volumes (if needed)
kubectl delete pvc -l app.kubernetes.io/name=bookkeeper
```

### Complete Cleanup

```bash
# Remove everything including metadata
helm uninstall my-bookkeeper
kubectl delete pvc -l app.kubernetes.io/name=bookkeeper
kubectl delete pv -l app.kubernetes.io/name=bookkeeper

# Clean ZooKeeper metadata (careful!)
kubectl exec zookeeper-0 -- /opt/zookeeper/bin/zkCli.sh deleteall /ledgers
```

## Advanced Configuration

### Custom Configuration File

```yaml
# values.yaml
bookkeeper:
  config:
    # Custom bookie configurations
    gcWaitTime: 300000
    journalSyncData: true
    journalAdaptiveGroupWrites: true
    entryLogFilePreallocationEnabled: true
    
    # Replication settings
    ensembleSize: 3
    writeQuorumSize: 2
    ackQuorumSize: 2
```

### Network Policies

```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: bookkeeper-network-policy
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: bookkeeper
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: pulsar-broker
    ports:
    - protocol: TCP
      port: 3181
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: zookeeper
    ports:
    - protocol: TCP
      port: 2181
```

## Support

For issues and questions:
- Apache BookKeeper Documentation: https://bookkeeper.apache.org/
- Kubernetes Documentation: https://kubernetes.io/docs/
- Helm Documentation: https://helm.sh/docs/

## License

This Helm chart is licensed under the Apache License 2.0.
