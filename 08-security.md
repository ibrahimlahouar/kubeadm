# 🔒 Sécurité dans Kubernetes

## 📍 Vue d'ensemble

La sécurité dans Kubernetes suit le principe de **défense en profondeur** avec plusieurs couches :

```
┌─────────────────────────────────────────────────┐
│           COUCHES DE SÉCURITÉ K8S               │
├─────────────────────────────────────────────────┤
│                                                 │
│  1. Infrastructure (Cloud/On-premise)           │
│  2. Cluster (API Server, etcd)                  │
│  3. Containers (Images, Runtime)                │
│  4. Code (Application)                          │
│  5. Réseau (NetworkPolicies, mTLS)             │
│  6. Données (Encryption at rest/in transit)     │
│                                                 │
└─────────────────────────────────────────────────┘
```

## 🔑 RBAC (Role-Based Access Control)

### Concepts RBAC

- **Role** : Permissions dans un namespace
- **ClusterRole** : Permissions au niveau cluster
- **RoleBinding** : Lie un Role à des utilisateurs/groupes
- **ClusterRoleBinding** : Lie un ClusterRole à des utilisateurs/groupes

### Exemple de Role et RoleBinding

```yaml
# Role : Développeur avec accès limité
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev-namespace
  name: developer-role
rules:
# Pods : lecture, création, mise à jour
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "create", "update", "patch"]
# Services : lecture seule
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list"]
# Deployments : tous les droits
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]

---
# RoleBinding : Assigner le role à un utilisateur
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: dev-namespace
subjects:
- kind: User
  name: john.doe@company.com
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRole et ClusterRoleBinding

```yaml
# ClusterRole : Lecture globale
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-reader
rules:
- apiGroups: [""]
  resources: ["nodes", "namespaces", "persistentvolumes"]
  verbs: ["get", "list"]
- apiGroups: ["apps"]
  resources: ["deployments", "daemonsets", "replicasets"]
  verbs: ["get", "list"]

---
# ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-reader-binding
subjects:
- kind: Group
  name: monitoring-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-reader
  apiGroup: rbac.authorization.k8s.io
```

### ServiceAccount et RBAC

```yaml
# ServiceAccount pour une application
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: production
  
---
# Role pour le ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: app-role
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["app-secret"] # Accès limité à un secret spécifique
  verbs: ["get"]

---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-role-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: app-service-account
  namespace: production
roleRef:
  kind: Role
  name: app-role
  apiGroup: rbac.authorization.k8s.io

---
# Pod utilisant le ServiceAccount
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
  namespace: production
spec:
  serviceAccountName: app-service-account
  containers:
  - name: app
    image: myapp:1.0
```

## 🛡️ Security Contexts

### Pod Security Context

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 1000          # UID de l'utilisateur
    runAsGroup: 3000         # GID du groupe
    fsGroup: 2000           # GID pour les volumes
    supplementalGroups: [4000] # Groupes supplémentaires
  containers:
  - name: app
    image: nginx
    securityContext:
      runAsNonRoot: true     # Empêche l'exécution en root
      readOnlyRootFilesystem: true # Système de fichiers en lecture seule
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL               # Supprime toutes les capabilities
        add:
        - NET_BIND_SERVICE  # Ajoute uniquement celle nécessaire
    volumeMounts:
    - name: cache
      mountPath: /tmp       # Pour écrire dans un fs read-only
  volumes:
  - name: cache
    emptyDir: {}
```

## 🚨 Pod Security Standards

### Niveaux de sécurité

1. **Privileged** : Non restreint
2. **Baseline** : Prévient les élévations de privilèges connues
3. **Restricted** : Pratiques de sécurité strictes

### Exemple de Pod Security Policy (PSP) - Deprecated

```yaml
# Utiliser Pod Security Standards à la place
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### Pod Security Admission

```yaml
# Configuration pour un namespace sécurisé
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Enforce : Bloque les pods non conformes
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: v1.25
    
    # Audit : Log les violations
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.25
    
    # Warn : Avertit lors de la création
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.25
```

## 🔐 Gestion des Secrets

### Types de Secrets

```yaml
# 1. Opaque (générique)
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  username: YWRtaW4=  # base64
  password: MWYyZDFlMmU2N2Rm  # base64

---
# 2. Docker Registry
apiVersion: v1
kind: Secret
metadata:
  name: docker-registry-secret
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: eyJhdXRocyI6eyJodHRwczovL2luZGV4L...

---
# 3. TLS Certificate
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0t...
  tls.key: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0t...

---
# 4. Service Account Token
apiVersion: v1
kind: Secret
metadata:
  name: sa-token
  annotations:
    kubernetes.io/service-account.name: my-sa
type: kubernetes.io/service-account-token
```

### Encryption at Rest

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key1
          secret: <BASE64_ENCODED_32_BYTE_KEY>
    - identity: {}
```

### Sealed Secrets

```bash
# Installation de Sealed Secrets
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.18.0/controller.yaml

# Créer un secret scellé
echo -n mypassword | kubectl create secret generic mysecret --dry-run=client --from-file=password=/dev/stdin -o yaml | kubeseal -o yaml > mysealedsecret.yaml
```

## 🌐 Network Security

### Network Policies (Exemples détaillés)

```yaml
# 1. Deny All - Point de départ Zero Trust
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# 2. Autoriser uniquement le trafic interne au namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector: {}
  egress:
  - to:
    - podSelector: {}
  # Autoriser DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    - podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53

---
# 3. Micro-segmentation pour une app 3-tiers
# Frontend Policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-netpol
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 8080

---
# Backend Policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-netpol
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
```

## 🔍 Audit et Compliance

### Configuration de l'audit

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Ne pas logger les requêtes de lecture sur les endpoints
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
    - group: ""
      resources: ["endpoints", "services"]

  # Ne pas logger les events des nodes
  - level: None
    userGroups: ["system:nodes"]
    verbs: ["get"]
    resources:
    - group: ""
      resources: ["nodes"]

  # Log les créations de pods au niveau Metadata
  - level: Metadata
    resources:
    - group: ""
      resources: ["pods"]
    verbs: ["create", "update", "patch"]

  # Log tout le reste au niveau RequestResponse
  - level: RequestResponse
```

## 🛠️ Outils de sécurité

### 1. Scanning de vulnérabilités

```bash
# Trivy pour scanner les images
trivy image myapp:latest

# Integration dans le pipeline CI/CD
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: trivy-scan
spec:
  template:
    spec:
      containers:
      - name: trivy
        image: aquasec/trivy:latest
        command:
        - trivy
        - image
        - --exit-code
        - "1"
        - --severity
        - "HIGH,CRITICAL"
        - myapp:latest
      restartPolicy: Never
EOF
```

### 2. Admission Controllers

```yaml
# Open Policy Agent (OPA) exemple
apiVersion: v1
kind: ConfigMap
metadata:
  name: opa-policies
  namespace: opa
data:
  policies.rego: |
    package kubernetes.admission
    
    deny[msg] {
      input.request.kind.kind == "Pod"
      input.request.object.spec.containers[_].image
      not starts_with(input.request.object.spec.containers[_].image, "registry.company.com/")
      msg := "Images must come from company registry"
    }
    
    deny[msg] {
      input.request.kind.kind == "Pod"
      not input.request.object.spec.securityContext.runAsNonRoot
      msg := "Pods must run as non-root"
    }
```

### 3. Falco - Runtime Security

```yaml
# Falco DaemonSet
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: falco
  namespace: falco
spec:
  selector:
    matchLabels:
      app: falco
  template:
    metadata:
      labels:
        app: falco
    spec:
      serviceAccountName: falco
      containers:
      - name: falco
        image: falcosecurity/falco:latest
        securityContext:
          privileged: true
        volumeMounts:
        - name: docker-socket
          mountPath: /host/var/run/docker.sock
        - name: proc-fs
          mountPath: /host/proc
        - name: boot-fs
          mountPath: /host/boot
        - name: lib-modules
          mountPath: /host/lib/modules
        - name: usr-fs
          mountPath: /host/usr
      volumes:
      - name: docker-socket
        hostPath:
          path: /var/run/docker.sock
      - name: proc-fs
        hostPath:
          path: /proc
      - name: boot-fs
        hostPath:
          path: /boot
      - name: lib-modules
        hostPath:
          path: /lib/modules
      - name: usr-fs
        hostPath:
          path: /usr
```

## 🎯 Bonnes pratiques de sécurité

### 1. Checklist de sécurité

```yaml
# ✅ Images
- Scanner toutes les images
- Utiliser des images minimales (distroless)
- Signer les images

# ✅ Pods
- Toujours définir securityContext
- Pas de privileged pods
- ReadOnlyRootFilesystem quand possible

# ✅ RBAC
- Principe du moindre privilège
- Pas de ClusterAdmin sauf nécessaire
- Audit régulier des permissions

# ✅ Secrets
- Encryption at rest
- Rotation régulière
- Jamais dans le code

# ✅ Network
- NetworkPolicies par défaut deny
- mTLS entre services (service mesh)
- Ingress avec TLS

# ✅ Monitoring
- Logs d'audit activés
- Alertes sur comportements anormaux
- Métriques de sécurité
```

### 2. Exemple de pod sécurisé

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: production
spec:
  serviceAccountName: app-sa
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: registry.company.com/app:1.0.0
    imagePullPolicy: Always
    ports:
    - containerPort: 8080
      protocol: TCP
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "200m"
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /app/cache
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

## 📝 Exercices pratiques

1. Configurez RBAC pour 3 équipes avec des accès différents
2. Créez des NetworkPolicies pour une application 3-tiers
3. Implémentez l'encryption at rest pour les secrets
4. Déployez Falco et créez des règles custom
5. Scannez et corrigez les vulnérabilités d'une image

---

[⬅️ Storage](./07-storage.md) | [🏠 Sommaire](./README.md) | [➡️ Monitoring](./09-monitoring.md) 