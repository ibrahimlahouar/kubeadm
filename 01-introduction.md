# 📖 Introduction à Kubernetes

## 🤔 Qu'est-ce que Kubernetes ?

Kubernetes (souvent abrégé **K8s**) est une plateforme open-source d'orchestration de conteneurs. En termes simples, c'est un système qui automatise le déploiement, la mise à l'échelle et la gestion des applications conteneurisées.

### Analogie simple
Imaginez Kubernetes comme un **chef d'orchestre** qui dirige un orchestre :
- Les **conteneurs** sont les musiciens
- Les **applications** sont les partitions
- **Kubernetes** est le chef qui s'assure que tout le monde joue en harmonie

## 🎯 Pourquoi utiliser Kubernetes ?

### Problèmes sans Kubernetes
1. **Gestion manuelle** : Démarrer/arrêter des conteneurs à la main
2. **Pas de haute disponibilité** : Si un conteneur crash, l'application est down
3. **Mise à l'échelle difficile** : Ajouter des instances manuellement
4. **Configuration complexe** : Gérer les liens entre conteneurs

### Solutions apportées par Kubernetes
✅ **Auto-healing** : Redémarre automatiquement les conteneurs qui crashent  
✅ **Scaling automatique** : Ajuste le nombre d'instances selon la charge  
✅ **Load balancing** : Distribue le trafic entre les conteneurs  
✅ **Gestion des secrets** : Stockage sécurisé des mots de passe  
✅ **Rolling updates** : Mises à jour sans interruption de service

## 🏗️ Concepts de base

### 1. Cluster
Un **cluster** est un ensemble de machines (physiques ou virtuelles) qui exécutent Kubernetes. Il comprend :
- **Master Node(s)** : Le cerveau qui prend les décisions
- **Worker Nodes** : Les machines qui exécutent vos applications

```
┌─────────────────────────────────────────────┐
│                  CLUSTER K8S                 │
├─────────────────────────────────────────────┤
│  ┌─────────────┐      ┌─────────────┐      │
│  │ Master Node │      │ Worker Node │      │
│  │   (Cerveau) │      │  (Exécuteur)│      │
│  └─────────────┘      └─────────────┘      │
│                       ┌─────────────┐      │
│                       │ Worker Node │      │
│                       │  (Exécuteur)│      │
│                       └─────────────┘      │
└─────────────────────────────────────────────┘
```

### 2. Pod
Un **Pod** est la plus petite unité déployable dans Kubernetes. C'est comme une "capsule" qui contient :
- Un ou plusieurs conteneurs
- Un stockage partagé
- Une adresse IP unique

```yaml
# Exemple simple d'un Pod
apiVersion: v1
kind: Pod
metadata:
  name: mon-premier-pod
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

### 3. Node
Un **Node** est une machine (physique ou virtuelle) dans votre cluster. Chaque node peut héberger plusieurs pods.

### 4. Namespace
Un **Namespace** est comme un "dossier virtuel" qui permet d'organiser et isoler les ressources dans un cluster.

## 🆚 Kubernetes vs Docker

| Aspect | Docker | Kubernetes |
|--------|--------|------------|
| **Rôle** | Crée et exécute des conteneurs | Orchestre des conteneurs |
| **Échelle** | Un conteneur à la fois | Gère des milliers de conteneurs |
| **Complexité** | Simple à apprendre | Plus complexe mais plus puissant |
| **Usage** | Développement local | Production à grande échelle |

### Relation Docker-Kubernetes
- **Docker** construit la maison (le conteneur)
- **Kubernetes** gère le quartier (l'ensemble des conteneurs)

## 📊 Architecture simplifiée

```
Votre MacBook                      Votre VPS
┌──────────────┐                ┌─────────────────┐
│              │                │  Cluster K8s    │
│   kubectl    │ ─────SSH────▶ │                 │
│              │                │  ┌───────────┐  │
│  Docker      │                │  │Master Node│  │
│  Desktop     │                │  └───────────┘  │
│              │                │                 │
└──────────────┘                │  ┌───────────┐  │
                                │  │Worker Node│  │
                                │  │           │  │
                                │  │  [Pod1]   │  │
                                │  │  [Pod2]   │  │
                                │  └───────────┘  │
                                └─────────────────┘
```

## 🎓 Points clés à retenir

1. **Kubernetes est un orchestrateur** : Il gère automatiquement vos conteneurs
2. **Haute disponibilité** : Vos applications restent accessibles même en cas de panne
3. **Scalabilité** : Ajustement automatique selon la charge
4. **Déclaratif** : Vous décrivez ce que vous voulez, K8s s'occupe du comment

## 🚀 Prochaine étape

Maintenant que vous comprenez les bases, passons à [l'architecture détaillée de Kubernetes](./02-architecture.md) pour comprendre comment tous ces éléments fonctionnent ensemble.

## 📝 Exercice rapide

1. Expliquez avec vos propres mots ce qu'est Kubernetes
2. Citez 3 avantages de Kubernetes par rapport à Docker seul
3. Qu'est-ce qu'un Pod ?

---

[⬅️ Retour au sommaire](./README.md) | [➡️ Architecture de Kubernetes](./02-architecture.md) 