# 🏛️ Architecture de Kubernetes

## 📐 Vue d'ensemble

L'architecture de Kubernetes suit un modèle **maître-esclave** (master-worker). Le cluster est divisé en deux parties principales :

```
┌────────────────────────────────────────────────────────────┐
│                     CLUSTER KUBERNETES                      │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  ┌─────────────────────────┐    ┌──────────────────────┐  │
│  │     CONTROL PLANE       │    │    DATA PLANE        │  │
│  │    (Master Node)        │    │   (Worker Nodes)     │  │
│  │                         │    │                      │  │
│  │  • API Server           │    │  • Pods              │  │
│  │  • etcd                 │    │  • Containers        │  │
│  │  • Scheduler            │    │  • Kubelet           │  │
│  │  • Controller Manager   │    │  • Kube-proxy        │  │
│  │  • Cloud Controller     │    │  • Container Runtime │  │
│  └─────────────────────────┘    └──────────────────────┘  │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

## 🧠 Control Plane (Master Node)

Le Control Plane est le cerveau de Kubernetes. Il prend toutes les décisions globales sur le cluster.

### Composants du Control Plane

```
┌─────────────────────────────────────────────────────┐
│                   CONTROL PLANE                      │
├─────────────────────────────────────────────────────┤
│                                                     │
│   ┌───────────────┐         ┌─────────────────┐   │
│   │   API Server  │◄────────┤      etcd       │   │
│   │  (Point       │         │  (Base de       │   │
│   │   d'entrée)   │         │   données)      │   │
│   └───────┬───────┘         └─────────────────┘   │
│           │                                         │
│           ▼                                         │
│   ┌───────────────┐         ┌─────────────────┐   │
│   │   Scheduler   │         │   Controller    │   │
│   │  (Placement   │         │    Manager      │   │
│   │   des pods)   │         │  (Surveillance) │   │
│   └───────────────┘         └─────────────────┘   │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 1. 🚪 API Server
**Rôle** : Point d'entrée unique pour toutes les opérations
- Reçoit les commandes kubectl
- Valide et traite les requêtes
- Met à jour l'état dans etcd

**Analogie** : C'est comme la réception d'un hôtel - toutes les demandes passent par là.

### 2. 💾 etcd
**Rôle** : Base de données distribuée qui stocke toute la configuration
- Stockage clé-valeur
- Hautement disponible
- Source de vérité du cluster

**Analogie** : C'est le registre central qui contient toutes les informations du cluster.

### 3. 📅 Scheduler
**Rôle** : Décide sur quel node placer les nouveaux pods
- Analyse les ressources disponibles
- Respecte les contraintes
- Optimise la distribution

**Analogie** : C'est comme un agent immobilier qui trouve le meilleur appartement pour chaque locataire.

### 4. 🎮 Controller Manager
**Rôle** : Exécute les boucles de contrôle
- Surveille l'état du cluster
- Effectue des actions correctives
- Maintient l'état désiré

**Analogie** : C'est le gardien qui s'assure que tout fonctionne comme prévu.

## 💪 Data Plane (Worker Nodes)

Les Worker Nodes sont les machines qui exécutent réellement vos applications.

```
┌─────────────────────────────────────────────────────┐
│                   WORKER NODE                        │
├─────────────────────────────────────────────────────┤
│                                                     │
│   ┌───────────────┐         ┌─────────────────┐   │
│   │    Kubelet    │         │   Kube-proxy    │   │
│   │  (Agent node) │         │    (Réseau)     │   │
│   └───────┬───────┘         └─────────────────┘   │
│           │                                         │
│           ▼                                         │
│   ┌─────────────────────────────────────────┐     │
│   │        Container Runtime (Docker)        │     │
│   └─────────────────────────────────────────┘     │
│                                                     │
│   ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│   │    Pod 1    │  │    Pod 2    │  │  Pod 3   │ │
│   │ ┌─────────┐ │  │ ┌─────────┐ │  │┌────────┐│ │
│   │ │Container│ │  │ │Container│ │  ││Container││ │
│   │ └─────────┘ │  │ └─────────┘ │  │└────────┘│ │
│   └─────────────┘  └─────────────┘  └──────────┘ │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 1. 🤖 Kubelet
**Rôle** : Agent qui s'exécute sur chaque node
- Communique avec l'API Server
- Gère les pods et leurs conteneurs
- Rapporte l'état du node

**Analogie** : C'est le concierge de l'immeuble qui s'occupe des appartements.

### 2. 🌐 Kube-proxy
**Rôle** : Gère les règles réseau sur chaque node
- Maintient les règles réseau
- Permet la communication entre services
- Load balancing local

**Analogie** : C'est le facteur qui s'assure que le courrier arrive à la bonne adresse.

### 3. 🐳 Container Runtime
**Rôle** : Exécute les conteneurs
- Docker, containerd, ou CRI-O
- Gère le cycle de vie des conteneurs
- Isole les processus

**Analogie** : C'est le moteur qui fait tourner les applications.

## 🔄 Flux de communication

Voici comment les composants interagissent lors du déploiement d'une application :

```
1. Utilisateur          2. API Server           3. etcd
   │                       │                       │
   │─── kubectl apply ────▶│                       │
   │                       │──── Stockage ────────▶│
   │                       │                       │
                           │                       
4. Scheduler            5. Controller            6. Kubelet
   │                       │                       │
   │◀─── Nouveau pod ──────│                       │
   │                       │                       │
   │─── Assigne node ─────▶│                       │
   │                       │                       │
   │                       │──── Crée pod ────────▶│
   │                       │                       │
                                                   │
                                          7. Container Runtime
                                                   │
                                                   ▼
                                            [Pod démarré]
```

## 🔐 Communication sécurisée

Tous les composants communiquent de manière sécurisée :

```
┌──────────────┐  HTTPS/TLS  ┌──────────────┐
│  API Server  │◄───────────►│   Kubelet    │
└──────────────┘             └──────────────┘
       │                             
       │         Certificats         
       │          Mutuels            
       ▼                             
┌──────────────┐             ┌──────────────┐
│     etcd     │             │  Kube-proxy  │
└──────────────┘             └──────────────┘
```

## 🏗️ Architecture Multi-Master (Haute Disponibilité)

Pour la production, on utilise plusieurs masters :

```
┌─────────────────────────────────────────────────────────┐
│                    CLUSTER HA                            │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│  │ Master 1 │  │ Master 2 │  │ Master 3 │            │
│  └──────────┘  └──────────┘  └──────────┘            │
│        │             │             │                    │
│        └─────────────┴─────────────┘                   │
│                      │                                  │
│               Load Balancer                             │
│                      │                                  │
│     ┌────────────────┴──────────────────┐             │
│     │                                    │             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │ Worker 1 │  │ Worker 2 │  │ Worker 3 │           │
│  └──────────┘  └──────────┘  └──────────┘           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## 🎯 Points clés à retenir

1. **Séparation des responsabilités** : Control Plane décide, Data Plane exécute
2. **Communication via API** : Tout passe par l'API Server
3. **État désiré vs État actuel** : K8s maintient toujours l'état désiré
4. **Haute disponibilité** : Architecture conçue pour la résilience

## 📊 Tableau récapitulatif

| Composant | Localisation | Rôle principal |
|-----------|--------------|----------------|
| API Server | Control Plane | Point d'entrée, validation |
| etcd | Control Plane | Stockage de configuration |
| Scheduler | Control Plane | Placement des pods |
| Controller Manager | Control Plane | Maintien de l'état |
| Kubelet | Worker Node | Gestion des pods |
| Kube-proxy | Worker Node | Gestion réseau |
| Container Runtime | Worker Node | Exécution des conteneurs |

## 🚀 Prochaine étape

Maintenant que vous comprenez l'architecture, explorons en détail [les composants principaux](./03-composants.md) et leur fonctionnement.

## 📝 Exercices

1. Dessinez l'architecture d'un cluster K8s avec 1 master et 2 workers
2. Expliquez le rôle de l'API Server dans vos propres mots
3. Que se passe-t-il si etcd tombe en panne ?
4. Pourquoi sépare-t-on Control Plane et Data Plane ?

---

[⬅️ Introduction](./01-introduction.md) | [🏠 Sommaire](./README.md) | [➡️ Les Composants](./03-composants.md) 