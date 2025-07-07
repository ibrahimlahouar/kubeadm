# 🔧 Les Composants Principaux de Kubernetes

## 📑 Table des matières
1. [Composants du Control Plane](#composants-du-control-plane)
2. [Composants des Worker Nodes](#composants-des-worker-nodes)
3. [Composants additionnels](#composants-additionnels)
4. [Interaction entre composants](#interaction-entre-composants)

## 🎛️ Composants du Control Plane

### 1. 🌐 API Server (kube-apiserver)

L'API Server est le **cœur** de Kubernetes. C'est le seul composant avec lequel vous interagissez directement.

#### Fonctionnalités principales
```
┌─────────────────────────────────────────┐
│            API SERVER                    │
├─────────────────────────────────────────┤
│                                         │
│  • Authentication (Qui êtes-vous?)      │
│  • Authorization (Que pouvez-vous faire?)│
│  • Admission Control (Est-ce valide?)   │
│  • Validation (Format correct?)         │
│  • Mutation (Modifications nécessaires?)│
│                                         │
└─────────────────────────────────────────┘
```

#### Exemple d'interaction
```bash
# Quand vous tapez cette commande :
kubectl get pods

# Voici ce qui se passe :
1. kubectl → API Server (HTTPS)
2. API Server vérifie votre identité
3. API Server vérifie vos permissions
4. API Server récupère les données depuis etcd
5. API Server renvoie la réponse à kubectl
```

#### Points clés
- **RESTful API** : Utilise HTTP/HTTPS
- **Stateless** : Ne stocke rien, tout est dans etcd
- **Extensible** : Peut ajouter des API custom
- **Watch mechanism** : Permet de surveiller les changements

### 2. 💾 etcd

etcd est une base de données **clé-valeur distribuée** qui stocke TOUTE la configuration du cluster.

#### Structure des données
```
/registry/
├── namespaces/
│   ├── default
│   ├── kube-system
│   └── my-app
├── pods/
│   ├── default/
│   │   ├── nginx-pod
│   │   └── mysql-pod
│   └── kube-system/
│       ├── kube-dns
│       └── kube-proxy
├── services/
├── deployments/
└── configmaps/
```

#### Caractéristiques importantes
- **Consensus RAFT** : Garantit la cohérence des données
- **Haute disponibilité** : Minimum 3 instances en production
- **Versioning** : Garde l'historique des modifications
- **Watch API** : Notifie les changements en temps réel

#### Commandes utiles
```bash
# Backup etcd (TRÈS IMPORTANT!)
ETCDCTL_API=3 etcdctl snapshot save backup.db

# Restore etcd
ETCDCTL_API=3 etcdctl snapshot restore backup.db
```

### 3. 📋 Scheduler (kube-scheduler)

Le Scheduler est comme un **agent de placement** qui trouve le meilleur node pour chaque pod.

#### Processus de scheduling
```
┌─────────────────────────────────────────────────┐
│              SCHEDULING PROCESS                  │
├─────────────────────────────────────────────────┤
│                                                 │
│  1. Pod créé sans node assigné                  │
│            ↓                                    │
│  2. Scheduler détecte le pod                    │
│            ↓                                    │
│  3. Filtre les nodes (Predicates)               │
│     • Ressources suffisantes?                  │
│     • Ports disponibles?                        │
│     • Volume attachable?                        │
│            ↓                                    │
│  4. Score les nodes (Priorities)                │
│     • Moins de charge                          │
│     • Affinité/Anti-affinité                   │
│     • Spread topologique                        │
│            ↓                                    │
│  5. Sélectionne le meilleur node               │
│            ↓                                    │
│  6. Bind le pod au node                         │
│                                                 │
└─────────────────────────────────────────────────┘
```

#### Exemple de contraintes
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  nodeSelector:
    gpu: "true"  # Seulement sur nodes avec GPU
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: gpu-pod
        topologyKey: kubernetes.io/hostname
  containers:
  - name: gpu-container
    image: tensorflow/tensorflow:latest-gpu
    resources:
      limits:
        nvidia.com/gpu: 1  # Demande 1 GPU
```

### 4. 🎯 Controller Manager (kube-controller-manager)

Le Controller Manager exécute plusieurs **contrôleurs** qui surveillent et maintiennent l'état du cluster.

#### Principaux contrôleurs
```
Controller Manager
├── Node Controller
│   └── Surveille la santé des nodes
├── Replication Controller
│   └── Maintient le nombre de réplicas
├── Endpoints Controller
│   └── Gère les endpoints des services
├── Service Account Controller
│   └── Crée les comptes de service
├── Deployment Controller
│   └── Gère les déploiements
├── StatefulSet Controller
│   └── Gère les applications stateful
└── Job Controller
    └── Gère les jobs batch
```

#### Boucle de réconciliation
```
while True:
    état_actuel = get_current_state()
    état_désiré = get_desired_state()
    
    if état_actuel != état_désiré:
        actions = calculate_actions(état_actuel, état_désiré)
        execute_actions(actions)
    
    sleep(sync_period)
```

## 🖥️ Composants des Worker Nodes

### 1. 🤖 Kubelet

Le Kubelet est l'**agent principal** sur chaque worker node.

#### Responsabilités
```
┌─────────────────────────────────────────────┐
│               KUBELET                        │
├─────────────────────────────────────────────┤
│                                             │
│  1. Pod Lifecycle Management                │
│     • Démarre/arrête les conteneurs        │
│     • Health checks (liveness/readiness)    │
│     • Rapporte le statut                   │
│                                             │
│  2. Resource Management                     │
│     • CPU/Memory limits                    │
│     • Volume mounting                      │
│     • Network setup                        │
│                                             │
│  3. Image Management                        │
│     • Pull des images                      │
│     • Garbage collection                   │
│     • Secret injection                     │
│                                             │
└─────────────────────────────────────────────┘
```

#### Configuration importante
```yaml
# /var/lib/kubelet/config.yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
evictionHard:
  memory.available: "100Mi"
  nodefs.available: "10%"
  imagefs.available: "15%"
maxPods: 110
```

### 2. 🌐 Kube-proxy

Kube-proxy gère la **connectivité réseau** pour les Services.

#### Modes de fonctionnement
```
┌─────────────────────────────────────────────┐
│           KUBE-PROXY MODES                   │
├─────────────────────────────────────────────┤
│                                             │
│  1. iptables mode (par défaut)             │
│     • Utilise iptables rules               │
│     • Performant                           │
│     • Round-robin load balancing           │
│                                             │
│  2. IPVS mode                               │
│     • Plus performant                      │
│     • Multiple algorithmes LB              │
│     • Nécessite IPVS kernel modules        │
│                                             │
│  3. userspace mode (legacy)                 │
│     • Proxy en userspace                   │
│     • Moins performant                     │
│     • Déprécié                             │
│                                             │
└─────────────────────────────────────────────┘
```

#### Exemple de règles iptables
```bash
# Service ClusterIP
-A KUBE-SERVICES -d 10.96.0.10/32 -p tcp -m tcp --dport 80 -j KUBE-SVC-NGINX

# Load balancing entre pods
-A KUBE-SVC-NGINX -m statistic --mode random --probability 0.5 -j KUBE-SEP-POD1
-A KUBE-SVC-NGINX -j KUBE-SEP-POD2
```

### 3. 🐳 Container Runtime

Le runtime de conteneur exécute les conteneurs.

#### Options disponibles
```
Container Runtimes
├── Docker
│   └── Le plus connu (deprecated dans K8s 1.24+)
├── containerd
│   └── Runtime léger, standard CNCF
├── CRI-O
│   └── Optimisé pour Kubernetes
└── Kata Containers
    └── Conteneurs avec isolation VM
```

#### Interface CRI
```
┌─────────────────────────────────────────────┐
│     Container Runtime Interface (CRI)        │
├─────────────────────────────────────────────┤
│                                             │
│  Kubelet  ←─── CRI API ──→  Runtime        │
│                                             │
│  • RuntimeService                           │
│    - CreateContainer()                      │
│    - StartContainer()                       │
│    - StopContainer()                        │
│                                             │
│  • ImageService                             │
│    - PullImage()                            │
│    - ListImages()                           │
│    - RemoveImage()                          │
│                                             │
└─────────────────────────────────────────────┘
```

## 🔌 Composants Additionnels

### 1. 📡 CoreDNS
Service DNS du cluster pour la résolution de noms.

```yaml
# Exemple de résolution DNS
my-service.my-namespace.svc.cluster.local
│          │            │   │        │
│          │            │   │        └── Domaine cluster
│          │            │   └── Type service
│          │            └── Namespace
│          └── Nom du service
└── Service
```

### 2. 🌍 Ingress Controller
Gère l'accès externe aux services.

```
Internet
    │
    ▼
┌──────────────┐
│Load Balancer │
└──────┬───────┘
       │
       ▼
┌──────────────┐     ┌─────────────┐
│   Ingress    │────▶│  Service 1  │────▶ [Pods]
│  Controller  │     └─────────────┘
│              │     ┌─────────────┐
│              │────▶│  Service 2  │────▶ [Pods]
└──────────────┘     └─────────────┘
```

### 3. 📊 Metrics Server
Collecte les métriques de ressources.

```bash
# Installation
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Utilisation
kubectl top nodes
kubectl top pods
```

## 🔄 Interaction entre Composants

### Flux de création d'un Deployment

```
┌─────────────────────────────────────────────────────────┐
│                  DEPLOYMENT FLOW                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. kubectl apply -f deployment.yaml                    │
│                 ↓                                       │
│  2. API Server valide et stocke dans etcd              │
│                 ↓                                       │
│  3. Deployment Controller détecte le nouveau deployment │
│                 ↓                                       │
│  4. Deployment Controller crée un ReplicaSet           │
│                 ↓                                       │
│  5. ReplicaSet Controller crée les Pods                │
│                 ↓                                       │
│  6. Scheduler assigne les Pods aux Nodes               │
│                 ↓                                       │
│  7. Kubelet démarre les conteneurs                     │
│                 ↓                                       │
│  8. Kube-proxy configure les règles réseau             │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## 🎯 Points clés à retenir

1. **Chaque composant a UN rôle spécifique**
2. **Communication via API Server uniquement**
3. **etcd est la source de vérité**
4. **Les contrôleurs maintiennent l'état désiré**
5. **Kubelet est le seul à toucher les conteneurs**

## 📝 Exercices pratiques

1. Listez tous les composants du control plane sur votre cluster
```bash
kubectl get pods -n kube-system
```

2. Vérifiez la santé des composants
```bash
kubectl get componentstatuses
```

3. Explorez la configuration du scheduler
```bash
kubectl get configmap -n kube-system kube-scheduler -o yaml
```

---

[⬅️ Architecture](./02-architecture.md) | [🏠 Sommaire](./README.md) | [➡️ Installation avec Kubeadm](./04-installation.md) 