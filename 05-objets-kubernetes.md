# 📦 Les Objets Kubernetes

## 🎯 Vue d'ensemble

Les objets Kubernetes sont des **entités persistantes** qui représentent l'état de votre cluster. Ils décrivent :
- Quelles applications conteneurisées sont en cours d'exécution
- Les ressources disponibles pour ces applications
- Les politiques de comportement (restart, upgrade, fault-tolerance)

## 🏗️ Structure d'un objet Kubernetes

Tous les objets Kubernetes suivent une structure similaire :

```yaml
apiVersion: v1                 # Version de l'API
kind: Pod                      # Type d'objet
metadata:                      # Métadonnées
  name: mon-pod
  namespace: default
  labels:
    app: mon-app
    env: dev
  annotations:
    description: "Mon premier pod"
spec:                          # Spécification (état désiré)
  containers:
  - name: nginx
    image: nginx:1.21
status:                        # État actuel (géré par K8s)
  phase: Running
  conditions:
  - type: Ready
    status: "True"
```

## 🌟 Pod - L'unité de base

### Qu'est-ce qu'un Pod ?

Un Pod est la plus petite unité déployable dans Kubernetes. Il encapsule :
- Un ou plusieurs conteneurs
- Stockage partagé (volumes)
- Une adresse IP unique
- Options de configuration

### Exemples de Pods

#### Pod simple
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

#### Pod multi-conteneurs
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
spec:
  containers:
  # Conteneur principal
  - name: app
    image: myapp:1.0
    ports:
    - containerPort: 8080
  
  # Sidecar pour les logs
  - name: log-aggregator
    image: fluentd:latest
    volumeMounts:
    - name: logs
      mountPath: /var/log
  
  volumes:
  - name: logs
    emptyDir: {}
```

### Patterns de Pods

```
┌─────────────────────────────────────────────────┐
│                  POD PATTERNS                    │
├─────────────────────────────────────────────────┤
│                                                 │
│  1. Sidecar Pattern                             │
│     ┌─────────────────────┐                    │
│     │  Main Container  │ Sidecar │              │
│     └─────────────────────┘                    │
│                                                 │
│  2. Ambassador Pattern                          │
│     ┌─────────────────────┐                    │
│     │  App  │ Ambassador │ → External Service  │
│     └─────────────────────┘                    │
│                                                 │
│  3. Adapter Pattern                             │
│     ┌─────────────────────┐                    │
│     │  App  │ Adapter │ → Monitoring           │
│     └─────────────────────┘                    │
│                                                 │
└─────────────────────────────────────────────────┘
```

## 🔄 ReplicaSet

### Rôle
Maintient un nombre stable de répliques de pods en cours d'exécution.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
spec:
  replicas: 3  # Nombre de répliques désirées
  selector:
    matchLabels:
      app: nginx
  template:    # Template de pod
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

### Commandes utiles
```bash
# Voir les ReplicaSets
kubectl get rs

# Scaler un ReplicaSet
kubectl scale rs nginx-replicaset --replicas=5

# Décrire un ReplicaSet
kubectl describe rs nginx-replicaset
```

## 🚀 Deployment

### Qu'est-ce qu'un Deployment ?

Un Deployment fournit des mises à jour déclaratives pour les Pods et ReplicaSets. Il gère :
- Le déploiement de nouvelles versions
- Le rollback en cas de problème
- La mise à l'échelle
- L'état de santé

### Exemple complet de Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deployment
  labels:
    app: webapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max pods en plus pendant update
      maxUnavailable: 1  # Max pods indisponibles
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
        version: v1.0
    spec:
      containers:
      - name: webapp
        image: myapp:1.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        env:
        - name: ENV
          value: "production"
        - name: DB_HOST
          value: "mysql-service"
```

### Stratégies de déploiement

```
┌─────────────────────────────────────────────────┐
│           DEPLOYMENT STRATEGIES                  │
├─────────────────────────────────────────────────┤
│                                                 │
│  1. RollingUpdate (Par défaut)                  │
│     v1 █████████░░░                            │
│     v2 ░░░░░░░░░███  Progressive              │
│                                                 │
│  2. Recreate                                    │
│     v1 █████████                               │
│     ↓  ░░░░░░░░░  Downtime                    │
│     v2 █████████                               │
│                                                 │
└─────────────────────────────────────────────────┘
```

### Gestion des versions

```bash
# Mettre à jour une image
kubectl set image deployment/webapp-deployment webapp=myapp:2.0

# Voir l'historique
kubectl rollout history deployment/webapp-deployment

# Rollback à la version précédente
kubectl rollout undo deployment/webapp-deployment

# Rollback à une version spécifique
kubectl rollout undo deployment/webapp-deployment --to-revision=2

# Pause/Resume un rollout
kubectl rollout pause deployment/webapp-deployment
kubectl rollout resume deployment/webapp-deployment
```

## 🌐 Service

### Types de Services

```yaml
# 1. ClusterIP (par défaut) - Accès interne uniquement
apiVersion: v1
kind: Service
metadata:
  name: webapp-clusterip
spec:
  type: ClusterIP
  selector:
    app: webapp
  ports:
  - port: 80        # Port du service
    targetPort: 8080 # Port du conteneur

---
# 2. NodePort - Accès externe via port du node
apiVersion: v1
kind: Service
metadata:
  name: webapp-nodeport
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080  # Port externe (30000-32767)

---
# 3. LoadBalancer - Accès externe via LB cloud
apiVersion: v1
kind: Service
metadata:
  name: webapp-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 8080

---
# 4. ExternalName - Alias DNS externe
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  type: ExternalName
  externalName: api.example.com
```

### Service Discovery

```
┌─────────────────────────────────────────────────┐
│            SERVICE DISCOVERY                     │
├─────────────────────────────────────────────────┤
│                                                 │
│  1. DNS                                         │
│     webapp-service.default.svc.cluster.local   │
│                                                 │
│  2. Environment Variables                       │
│     WEBAPP_SERVICE_HOST=10.96.0.10            │
│     WEBAPP_SERVICE_PORT=80                     │
│                                                 │
│  3. Headless Service (pas de ClusterIP)        │
│     Retourne les IPs des pods directement      │
│                                                 │
└─────────────────────────────────────────────────┘
```

## 🔐 ConfigMap et Secret

### ConfigMap - Configuration non sensible

```yaml
# Création via YAML
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.properties: |
    database.url=jdbc:mysql://mysql:3306/mydb
    app.name=MyApplication
    log.level=INFO
  
  nginx.conf: |
    server {
      listen 80;
      server_name example.com;
      location / {
        proxy_pass http://backend;
      }
    }

---
# Utilisation dans un Pod
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    # Variable d'environnement depuis ConfigMap
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: log.level
    
    # Monter comme volume
    volumeMounts:
    - name: config
      mountPath: /etc/config
  
  volumes:
  - name: config
    configMap:
      name: app-config
```

### Secret - Données sensibles

```yaml
# Création d'un Secret
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  # Les valeurs doivent être encodées en base64
  username: YWRtaW4=     # echo -n "admin" | base64
  password: cGFzc3dvcmQ= # echo -n "password" | base64

---
# Utilisation dans un Pod
apiVersion: v1
kind: Pod
metadata:
  name: db-pod
spec:
  containers:
  - name: db
    image: mysql:5.7
    env:
    - name: MYSQL_ROOT_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
```

### Commandes pour ConfigMap et Secret

```bash
# Créer un ConfigMap depuis des fichiers
kubectl create configmap nginx-config --from-file=nginx.conf

# Créer un Secret depuis la ligne de commande
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password='S3cur3P@ss'

# Voir les ConfigMaps/Secrets
kubectl get configmaps
kubectl get secrets

# Décrire (sans voir les valeurs des secrets)
kubectl describe secret db-secret
```

## 💾 Volumes

### Types de volumes courants

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-demo
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    # 1. emptyDir - Ephémère, vit avec le pod
    - name: cache
      mountPath: /cache
    
    # 2. hostPath - Répertoire de l'hôte
    - name: logs
      mountPath: /var/log/app
    
    # 3. PersistentVolumeClaim
    - name: data
      mountPath: /data
    
    # 4. ConfigMap comme volume
    - name: config
      mountPath: /etc/config
    
    # 5. Secret comme volume
    - name: secrets
      mountPath: /etc/secrets
      readOnly: true
  
  volumes:
  - name: cache
    emptyDir: {}
  
  - name: logs
    hostPath:
      path: /var/log/apps
      type: DirectoryOrCreate
  
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
  
  - name: config
    configMap:
      name: app-config
  
  - name: secrets
    secret:
      secretName: app-secrets
      defaultMode: 0400  # Permissions
```

## 🏷️ Labels et Selectors

### Labels - Métadonnées clé-valeur

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: labeled-pod
  labels:
    app: webapp
    tier: frontend
    environment: production
    version: v1.2.3
    release: stable
```

### Selectors - Requêtes sur les labels

```bash
# Sélection par égalité
kubectl get pods -l app=webapp
kubectl get pods -l environment=production,tier=frontend

# Sélection par ensemble
kubectl get pods -l 'environment in (production, staging)'
kubectl get pods -l 'tier notin (backend)'
kubectl get pods -l 'version'  # A le label version

# Dans les objets YAML
selector:
  matchLabels:
    app: webapp
  matchExpressions:
  - {key: tier, operator: In, values: [frontend, api]}
  - {key: environment, operator: NotIn, values: [dev]}
```

## 📊 Tableau récapitulatif

| Objet | Utilisation | Gère |
|-------|-------------|------|
| **Pod** | Unité de base | Conteneurs |
| **ReplicaSet** | Maintenir N répliques | Pods |
| **Deployment** | Déploiements déclaratifs | ReplicaSets |
| **Service** | Exposition réseau | Endpoints |
| **ConfigMap** | Configuration | Données non-sensibles |
| **Secret** | Données sensibles | Credentials, certs |
| **Volume** | Stockage | Données persistantes |

## 🎯 Bonnes pratiques

1. **Toujours utiliser des Deployments** plutôt que des Pods nus
2. **Définir des ressources** (requests/limits) pour tous les conteneurs
3. **Utiliser des health checks** (liveness/readiness probes)
4. **Séparer configuration** (ConfigMap) et secrets (Secret)
5. **Labels cohérents** pour faciliter la gestion
6. **Namespaces** pour isoler les environnements

## 📝 Exercices pratiques

1. Créez un Deployment avec 3 répliques d'une app Node.js
2. Exposez-le via un Service NodePort
3. Créez un ConfigMap avec la configuration de l'app
4. Montez un Secret contenant les credentials de DB
5. Effectuez un rolling update vers une nouvelle version

---

[⬅️ Installation](./04-installation.md) | [🏠 Sommaire](./README.md) | [➡️ Networking](./06-networking.md) 