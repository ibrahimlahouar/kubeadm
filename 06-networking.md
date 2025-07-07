# 🌐 Le Réseau dans Kubernetes

## 📍 Vue d'ensemble

Le modèle réseau de Kubernetes repose sur plusieurs principes fondamentaux :

1. **Chaque Pod a une IP unique** dans le cluster
2. **Tous les Pods peuvent communiquer** entre eux sans NAT
3. **Tous les Nodes peuvent communiquer** avec tous les Pods sans NAT
4. **L'IP qu'un Pod voit de lui-même** est la même que celle vue par les autres

## 🏗️ Architecture réseau

```
┌─────────────────────────────────────────────────────┐
│                 CLUSTER NETWORK                      │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Node 1 (192.168.1.10)        Node 2 (192.168.1.11)│
│  ┌──────────────────┐         ┌──────────────────┐ │
│  │ Pod Network      │         │ Pod Network      │ │
│  │ 10.244.1.0/24    │         │ 10.244.2.0/24    │ │
│  │                  │         │                  │ │
│  │ ┌─────┐ ┌─────┐ │         │ ┌─────┐ ┌─────┐ │ │
│  │ │Pod A│ │Pod B│ │         │ │Pod C│ │Pod D│ │ │
│  │ │.1.2 │ │.1.3 │ │◄────────┤►│.2.2 │ │.2.3 │ │ │
│  │ └─────┘ └─────┘ │         │ └─────┘ └─────┘ │ │
│  └──────────────────┘         └──────────────────┘ │
│                                                     │
│  Service Network: 10.96.0.0/12                      │
│  DNS Service: 10.96.0.10                           │
│                                                     │
└─────────────────────────────────────────────────────┘
```

## 🔌 CNI (Container Network Interface)

### Qu'est-ce que CNI ?

CNI est la spécification qui définit comment les plugins réseau configurent la connectivité réseau des conteneurs.

### Plugins CNI populaires

```yaml
# 1. Flannel - Simple, overlay network
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# 2. Calico - Riche en fonctionnalités, supporte NetworkPolicies
kubectl create -f https://projectcalico.docs.tigera.io/manifests/tigera-operator.yaml

# 3. Weave Net - Simple à installer
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

# 4. Cilium - eBPF-based, très performant
kubectl create -f https://raw.githubusercontent.com/cilium/cilium/v1.12/install/kubernetes/quick-install.yaml
```

### Comparaison des CNI

| CNI | Mode | Performance | NetworkPolicy | Complexité |
|-----|------|------------|---------------|------------|
| Flannel | Overlay (VXLAN) | Moyenne | Non | Faible |
| Calico | BGP/IPIP | Haute | Oui | Moyenne |
| Weave | Overlay | Moyenne | Oui | Faible |
| Cilium | eBPF | Très haute | Oui+ | Haute |

## 🌍 Services - Exposition des applications

### Types de Services détaillés

#### 1. ClusterIP

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP  # Par défaut
  selector:
    app: backend
  ports:
  - name: http
    port: 80        # Port du service
    targetPort: 8080 # Port du conteneur
    protocol: TCP
```

```
┌─────────────────────────────────────┐
│         ClusterIP Service           │
├─────────────────────────────────────┤
│                                     │
│  Client Pod ──► Service VIP ──┐    │
│                 (10.96.0.100)  │    │
│                               ▼    │
│                         ┌──────────┐│
│                         │ iptables ││
│                         │  rules   ││
│                         └──────────┘│
│                               │     │
│              ┌────────────────┼───┐ │
│              ▼                ▼   ▼ │
│         Backend Pod      Pod    Pod │
│                                     │
└─────────────────────────────────────┘
```

#### 2. NodePort

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080  # 30000-32767
```

```
External Client
      │
      ▼
Node IP:30080
      │
      ▼
┌─────────────────────────┐
│    kube-proxy           │
│  (iptables rules)       │
└─────────────────────────┘
      │
      ▼
Service (ClusterIP)
      │
      ▼
Target Pods
```

#### 3. LoadBalancer

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-lb
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

```
Internet
    │
    ▼
Cloud Load Balancer
(External IP)
    │
    ▼
NodePort on all nodes
    │
    ▼
Service ClusterIP
    │
    ▼
Pod endpoints
```

### Session Affinity

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sticky-service
spec:
  selector:
    app: web
  sessionAffinity: ClientIP  # Sessions collantes
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3 heures
  ports:
  - port: 80
    targetPort: 8080
```

## 🚪 Ingress

### Qu'est-ce qu'un Ingress ?

L'Ingress est un objet API qui gère l'accès externe aux services, typiquement HTTP/HTTPS.

### Exemple d'Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### Architecture Ingress

```
┌─────────────────────────────────────────────────┐
│                 INGRESS FLOW                     │
├─────────────────────────────────────────────────┤
│                                                 │
│  Internet                                       │
│     │                                          │
│     ▼                                          │
│  DNS (app.example.com → LoadBalancer IP)       │
│     │                                          │
│     ▼                                          │
│  Cloud LoadBalancer                            │
│     │                                          │
│     ▼                                          │
│  Ingress Controller (nginx-ingress pods)       │
│     │                                          │
│     ├─── /api ──────► api-service:8080        │
│     │                      │                   │
│     │                      ▼                   │
│     │                  API Pods                │
│     │                                          │
│     └─── / ─────────► frontend-service:80     │
│                            │                   │
│                            ▼                   │
│                      Frontend Pods             │
│                                                 │
└─────────────────────────────────────────────────┘
```

### Ingress Controllers

```bash
# NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.5.1/deploy/static/provider/cloud/deploy.yaml

# Traefik
helm install traefik traefik/traefik

# HAProxy
helm install haproxy haproxytech/kubernetes-ingress

# Contour
kubectl apply -f https://projectcontour.io/quickstart/contour.yaml
```

## 🔒 Network Policies

### Qu'est-ce qu'une Network Policy ?

Les Network Policies sont des règles qui contrôlent le trafic réseau entre les pods.

### Exemples de Network Policies

#### 1. Deny all (Zero Trust)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}  # Tous les pods
  policyTypes:
  - Ingress
  - Egress
```

#### 2. Autoriser uniquement depuis un namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-app-namespace
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: app-namespace
    - podSelector:
        matchLabels:
          role: backend
    ports:
    - protocol: TCP
      port: 5432
```

#### 3. Politique complexe

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-netpol
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Autoriser depuis l'ingress controller
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
  egress:
  # Autoriser vers la base de données
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
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
  # Autoriser vers l'extérieur (APIs)
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8
        - 192.168.0.0/16
    ports:
    - protocol: TCP
      port: 443
```

## 📡 Service Mesh

### Qu'est-ce qu'un Service Mesh ?

Un service mesh est une couche d'infrastructure dédiée pour gérer la communication service-à-service.

### Istio - Exemple de configuration

```yaml
# VirtualService
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: webapp-routing
spec:
  hosts:
  - webapp
  http:
  - match:
    - headers:
        version:
          exact: v2
    route:
    - destination:
        host: webapp
        subset: v2
  - route:
    - destination:
        host: webapp
        subset: v1
      weight: 90
    - destination:
        host: webapp
        subset: v2
      weight: 10

---
# DestinationRule
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: webapp-destination
spec:
  host: webapp
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
    loadBalancer:
      simple: LEAST_REQUEST
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

## 🔍 DNS dans Kubernetes

### CoreDNS

```yaml
# Service DNS
<service>.<namespace>.svc.cluster.local

# Pod DNS
<pod-ip>.<namespace>.pod.cluster.local

# Exemples:
# Service: webapp-service.production.svc.cluster.local
# Pod: 10-244-1-5.production.pod.cluster.local
```

### Configuration DNS personnalisée

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  example.server: |
    example.com:53 {
        forward . 8.8.8.8
    }
```

## 🛠️ Debugging réseau

### Outils de diagnostic

```bash
# 1. Test de connectivité
kubectl run test-pod --image=busybox -it --rm -- sh
# Dans le pod:
nslookup kubernetes.default
wget -O- http://service-name

# 2. Voir les endpoints
kubectl get endpoints

# 3. Logs kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy

# 4. Tcpdump dans un pod
kubectl exec -it pod-name -- tcpdump -i eth0

# 5. Network policies appliquées
kubectl get networkpolicies -A
```

### Pod de debug réseau

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: netshoot
spec:
  containers:
  - name: netshoot
    image: nicolaka/netshoot
    command: ["sleep", "3600"]
```

## 📊 Monitoring réseau

### Métriques importantes

```yaml
# Prometheus metrics
- container_network_receive_bytes_total
- container_network_transmit_bytes_total
- container_network_receive_errors_total
- container_network_transmit_errors_total
```

## 🎯 Bonnes pratiques

1. **Utilisez des Network Policies** pour la sécurité Zero Trust
2. **Préférez les Services aux IPs de pods** (qui changent)
3. **Utilisez Ingress** pour l'exposition HTTP/HTTPS
4. **Monitoring du réseau** est crucial en production
5. **Service Mesh** pour des besoins avancés (observabilité, sécurité)

## 📝 Exercices pratiques

1. Déployez une application avec frontend/backend
2. Exposez le frontend via Ingress avec TLS
3. Créez des Network Policies pour isoler les composants
4. Testez la connectivité avec des pods de debug
5. Configurez un service mesh basique

---

[⬅️ Les Objets](./05-objets-kubernetes.md) | [🏠 Sommaire](./README.md) | [➡️ Storage](./07-storage.md) 