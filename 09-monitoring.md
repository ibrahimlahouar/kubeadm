# 📊 Monitoring et Observabilité dans Kubernetes

## 📍 Vue d'ensemble

L'observabilité dans Kubernetes repose sur trois piliers :

```
┌─────────────────────────────────────────────────┐
│           PILIERS DE L'OBSERVABILITÉ            │
├─────────────────────────────────────────────────┤
│                                                 │
│  📊 MÉTRIQUES          📝 LOGS         🔍 TRACES │
│  (Prometheus)      (ELK/Loki)      (Jaeger)    │
│                                                 │
│  • CPU/Memory      • App logs      • Requests  │
│  • Requests/s      • System logs   • Latency   │
│  • Error rates     • Events        • Dependencies│
│                                                 │
└─────────────────────────────────────────────────┘
```

## 🎯 Métriques avec Prometheus

### Architecture Prometheus

```
┌─────────────────────────────────────────────────┐
│              PROMETHEUS STACK                    │
├─────────────────────────────────────────────────┤
│                                                 │
│  Kubernetes Cluster                             │
│  ┌─────────────┐    ┌─────────────┐           │
│  │   Pods      │    │   Nodes     │           │
│  │ /metrics    │    │ node-exporter│          │
│  └──────┬──────┘    └──────┬──────┘           │
│         │                   │                   │
│         └─────────┬─────────┘                  │
│                   ▼                             │
│         ┌─────────────────┐                    │
│         │   Prometheus    │                    │
│         │    Server       │                    │
│         └────────┬────────┘                    │
│                  │                              │
│         ┌────────┴────────┐                    │
│         ▼                 ▼                    │
│    ┌─────────┐      ┌──────────┐              │
│    │ Grafana │      │AlertManager│             │
│    └─────────┘      └──────────┘              │
│                                                 │
└─────────────────────────────────────────────────┘
```

### Installation avec Helm

```bash
# Ajouter le repo Prometheus
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Installer Prometheus Stack (inclut Grafana)
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=30d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=fast-ssd \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi
```

### Configuration Prometheus

```yaml
# prometheus-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    
    rule_files:
      - /etc/prometheus/rules/*.yml
    
    scrape_configs:
      # Kubernetes API Server
      - job_name: 'kubernetes-apiservers'
        kubernetes_sd_configs:
        - role: endpoints
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: default;kubernetes;https
      
      # Pods avec annotations
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
        - role: pod
        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
          action: replace
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
          target_label: __address__
```

### ServiceMonitor pour scraper une application

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: app-metrics
  namespace: monitoring
  labels:
    app: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

### Application avec métriques Prometheus

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-metrics
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: app
        image: myapp:1.0
        ports:
        - name: http
          containerPort: 8080
        - name: metrics
          containerPort: 8080
        env:
        - name: METRICS_PORT
          value: "8080"
```

## 📈 Grafana Dashboards

### Dashboard Kubernetes personnalisé

```json
{
  "dashboard": {
    "title": "Kubernetes Cluster Overview",
    "panels": [
      {
        "title": "CPU Usage by Namespace",
        "targets": [
          {
            "expr": "sum(rate(container_cpu_usage_seconds_total{container!=\"\"}[5m])) by (namespace)"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Memory Usage by Pod",
        "targets": [
          {
            "expr": "sum(container_memory_working_set_bytes{container!=\"\"}) by (pod)"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Network I/O",
        "targets": [
          {
            "expr": "sum(rate(container_network_receive_bytes_total[5m])) by (pod)"
          }
        ],
        "type": "graph"
      }
    ]
  }
}
```

### Alertes Prometheus

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: app-alerts
  namespace: monitoring
spec:
  groups:
  - name: app.rules
    interval: 30s
    rules:
    # Pod crashant
    - alert: PodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} crash en boucle"
        description: "Le pod {{ $labels.pod }} a redémarré {{ $value }} fois en 15 minutes"
    
    # Haute utilisation CPU
    - alert: HighCPUUsage
      expr: |
        sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod, namespace)
        / sum(kube_pod_container_resource_limits{resource="cpu"}) by (pod, namespace) > 0.8
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Haute utilisation CPU pour {{ $labels.pod }}"
        description: "Le pod utilise plus de 80% de sa limite CPU"
    
    # Mémoire élevée
    - alert: HighMemoryUsage
      expr: |
        sum(container_memory_working_set_bytes{container!=""}) by (pod, namespace)
        / sum(kube_pod_container_resource_limits{resource="memory"}) by (pod, namespace) > 0.9
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Haute utilisation mémoire pour {{ $labels.pod }}"
    
    # PVC presque plein
    - alert: PersistentVolumeFillingUp
      expr: |
        kubelet_volume_stats_available_bytes / kubelet_volume_stats_capacity_bytes < 0.1
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "PVC {{ $labels.persistentvolumeclaim }} presque plein"
```

## 📝 Logs avec la Stack ELK

### Architecture ELK/EFK

```
┌─────────────────────────────────────────────────┐
│                 EFK STACK                        │
├─────────────────────────────────────────────────┤
│                                                 │
│  Nodes                                          │
│  ┌──────────────┐                              │
│  │  Fluentd     │ (DaemonSet)                  │
│  │  /var/log/*  │                              │
│  └──────┬───────┘                              │
│         │                                       │
│         ▼                                       │
│  ┌──────────────┐                              │
│  │Elasticsearch │                              │
│  │  (Storage)   │                              │
│  └──────┬───────┘                              │
│         │                                       │
│         ▼                                       │
│  ┌──────────────┐                              │
│  │   Kibana     │                              │
│  │(Visualization)│                              │
│  └──────────────┘                              │
│                                                 │
└─────────────────────────────────────────────────┘
```

### Installation d'Elasticsearch

```yaml
# elasticsearch-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
  namespace: logging
spec:
  serviceName: elasticsearch
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      initContainers:
      - name: increase-vm-max-map
        image: busybox
        command: ["sysctl", "-w", "vm.max_map_count=262144"]
        securityContext:
          privileged: true
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:7.17.0
        ports:
        - containerPort: 9200
          name: http
        - containerPort: 9300
          name: transport
        env:
        - name: cluster.name
          value: k8s-logs
        - name: node.name
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: discovery.seed_hosts
          value: "elasticsearch-0.elasticsearch,elasticsearch-1.elasticsearch,elasticsearch-2.elasticsearch"
        - name: cluster.initial_master_nodes
          value: "elasticsearch-0,elasticsearch-1,elasticsearch-2"
        - name: ES_JAVA_OPTS
          value: "-Xms512m -Xmx512m"
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 30Gi
```

### Fluentd DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      serviceAccountName: fluentd
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1-debian-elasticsearch
        env:
        - name: FLUENT_ELASTICSEARCH_HOST
          value: "elasticsearch.logging.svc.cluster.local"
        - name: FLUENT_ELASTICSEARCH_PORT
          value: "9200"
        - name: FLUENT_ELASTICSEARCH_SCHEME
          value: "http"
        - name: FLUENTD_SYSTEMD_CONF
          value: disable
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: config
          mountPath: /fluentd/etc
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: config
        configMap:
          name: fluentd-config
```

### Configuration Fluentd

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: logging
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/containers/*.log
      pos_file /var/log/fluentd-containers.log.pos
      tag kubernetes.*
      read_from_head true
      <parse>
        @type json
        time_format %Y-%m-%dT%H:%M:%S.%NZ
      </parse>
    </source>
    
    <filter kubernetes.**>
      @type kubernetes_metadata
      @id filter_kube_metadata
    </filter>
    
    <filter kubernetes.**>
      @type parser
      key_name log
      <parse>
        @type json
      </parse>
    </filter>
    
    <match **>
      @type elasticsearch
      host elasticsearch.logging.svc.cluster.local
      port 9200
      logstash_format true
      logstash_prefix k8s
      <buffer>
        @type file
        path /var/log/fluentd-buffers/kubernetes.system.buffer
        flush_mode interval
        retry_type exponential_backoff
        flush_thread_count 2
        flush_interval 5s
        retry_forever
        retry_max_interval 30
        chunk_limit_size 2M
        total_limit_size 500M
        overflow_action block
      </buffer>
    </match>
```

## 🔍 Tracing avec Jaeger

### Installation de Jaeger

```yaml
# jaeger-operator.yaml
kubectl create namespace observability
kubectl create -f https://github.com/jaegertracing/jaeger-operator/releases/download/v1.41.0/jaeger-operator.yaml -n observability

# jaeger-instance.yaml
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger
  namespace: observability
spec:
  strategy: production
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: http://elasticsearch.logging.svc:9200
```

### Application avec tracing

```python
# app.py avec OpenTelemetry
from opentelemetry import trace
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

# Configuration du tracer
resource = Resource(attributes={
    "service.name": "my-app"
})

provider = TracerProvider(resource=resource)
processor = BatchSpanProcessor(
    JaegerExporter(
        agent_host_name="jaeger-agent.observability.svc.cluster.local",
        agent_port=6831,
    )
)
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)
tracer = trace.get_tracer(__name__)

# Utilisation dans l'application
@app.route('/api/users/<user_id>')
def get_user(user_id):
    with tracer.start_as_current_span("get_user") as span:
        span.set_attribute("user.id", user_id)
        
        # Appel base de données
        with tracer.start_as_current_span("db_query"):
            user = db.get_user(user_id)
        
        # Appel service externe
        with tracer.start_as_current_span("external_api_call"):
            enriched_data = external_api.get_data(user_id)
        
        return jsonify(user)
```

## 🎮 Kubernetes Dashboard

### Installation

```bash
# Installer le dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

# Créer un compte admin
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF

# Obtenir le token
kubectl -n kubernetes-dashboard create token admin-user
```

## 📱 Alerting avec AlertManager

### Configuration AlertManager

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml: |
    global:
      resolve_timeout: 5m
      slack_api_url: 'YOUR_SLACK_WEBHOOK_URL'
    
    route:
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 12h
      receiver: 'default'
      routes:
      - match:
          severity: critical
        receiver: pagerduty
      - match:
          severity: warning
        receiver: slack
    
    receivers:
    - name: 'default'
      slack_configs:
      - channel: '#alerts'
        title: 'Kubernetes Alert'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
    
    - name: 'slack'
      slack_configs:
      - channel: '#k8s-warnings'
        send_resolved: true
    
    - name: 'pagerduty'
      pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_KEY'
```

## 🔧 Métriques personnalisées

### Horizontal Pod Autoscaler avec métriques custom

```yaml
# Application exposant des métriques custom
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-custom-metrics
spec:
  replicas: 2
  selector:
    matchLabels:
      app: custom-app
  template:
    metadata:
      labels:
        app: custom-app
    spec:
      containers:
      - name: app
        image: myapp:1.0
        ports:
        - containerPort: 8080
        - containerPort: 9090  # Métriques

---
# HPA avec métriques custom
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: custom-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app-with-custom-metrics
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
```

## 📊 Stack complète de monitoring

### Exemple de déploiement complet

```bash
#!/bin/bash
# deploy-monitoring.sh

# Créer les namespaces
kubectl create namespace monitoring
kubectl create namespace logging

# Installer Prometheus + Grafana
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword=admin123

# Installer ELK
kubectl apply -f elasticsearch-statefulset.yaml
kubectl apply -f kibana-deployment.yaml
kubectl apply -f fluentd-daemonset.yaml

# Installer Jaeger
kubectl apply -f https://github.com/jaegertracing/jaeger-operator/releases/download/v1.41.0/jaeger-operator.yaml

# Configurer les ingress
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: monitoring-ingress
  namespace: monitoring
spec:
  rules:
  - host: grafana.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus-grafana
            port:
              number: 80
  - host: prometheus.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus-kube-prometheus-prometheus
            port:
              number: 9090
EOF
```

## 🎯 Bonnes pratiques

1. **Retention des données** : Configurez selon vos besoins
2. **Cardinality** : Évitez trop de labels uniques
3. **Sampling** : Pour le tracing en production
4. **Dashboards** : Créez des vues par équipe/service
5. **Alertes** : Commencez simple, affinez progressivement
6. **SLO/SLI** : Définissez des objectifs mesurables

## 📝 Exercices pratiques

1. Déployez la stack Prometheus/Grafana
2. Créez des dashboards pour votre application
3. Configurez des alertes pour CPU/Memory/Pods
4. Implémentez le tracing distribué
5. Créez un dashboard unifié logs/metrics/traces

---

[⬅️ Sécurité](./08-security.md) | [🏠 Sommaire](./README.md) | [➡️ Exercices Pratiques](./10-pratique.md) 