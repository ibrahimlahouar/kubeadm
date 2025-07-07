# 🚀 Documentation Complète Kubernetes avec Kubeadm

> Une documentation progressive pour maîtriser Kubernetes de zéro à expert

## 📋 Table des matières

### 🎯 Niveau Débutant
1. [Introduction à Kubernetes](./01-introduction.md)
   - Qu'est-ce que Kubernetes ?
   - Pourquoi utiliser Kubernetes ?
   - Concepts de base
   - Comparaison avec Docker

2. [Architecture de Kubernetes](./02-architecture.md)
   - Vue d'ensemble de l'architecture
   - Master Node (Control Plane)
   - Worker Nodes
   - Communication entre composants

3. [Les Composants Principaux](./03-composants.md)
   - API Server
   - etcd
   - Scheduler
   - Controller Manager
   - Kubelet
   - Kube-proxy

### 🔧 Niveau Intermédiaire
4. [Installation avec Kubeadm](./04-installation.md)
   - Prérequis système
   - Installation sur votre VPS
   - Configuration du cluster
   - Connexion depuis votre MacBook

5. [Les Objets Kubernetes](./05-objets-kubernetes.md)
   - Pods
   - Services
   - Deployments
   - ReplicaSets
   - ConfigMaps & Secrets
   - Volumes

6. [Le Réseau dans Kubernetes](./06-networking.md)
   - Modèle réseau de K8s
   - Services et LoadBalancers
   - Ingress Controllers
   - Network Policies

### 🚀 Niveau Avancé
7. [Stockage Persistant](./07-storage.md)
   - Volumes persistants
   - StorageClasses
   - StatefulSets
   - CSI (Container Storage Interface)

8. [Sécurité dans Kubernetes](./08-security.md)
   - RBAC (Role-Based Access Control)
   - ServiceAccounts
   - Security Contexts
   - Network Policies

9. [Monitoring et Observabilité](./09-monitoring.md)
   - Prometheus & Grafana
   - Logs avec ELK Stack
   - Métriques et alertes
   - Debugging des applications

### 💪 Pratique
10. [Exercices Pratiques](./10-pratique.md)
    - Déployer une application web
    - Mise à l'échelle automatique
    - CI/CD avec Kubernetes
    - Projets complets

## 🖥️ Votre Infrastructure

Avec vos spécifications (500 GB SSD, 48 GB RAM, 12 cores), vous avez une excellente configuration pour :
- Créer un cluster multi-nodes
- Tester des applications complexes
- Simuler des environnements de production
- Expérimenter avec différentes configurations

## 🎯 Objectifs d'apprentissage

À la fin de cette documentation, vous serez capable de :
- ✅ Comprendre l'architecture complète de Kubernetes
- ✅ Installer et configurer un cluster avec kubeadm
- ✅ Déployer des applications depuis votre MacBook vers votre VPS
- ✅ Gérer la sécurité, le réseau et le stockage
- ✅ Monitorer et débugger vos applications
- ✅ Implémenter des pipelines CI/CD

## 🚦 Comment utiliser cette documentation

1. **Commencez par l'introduction** pour comprendre les concepts de base
2. **Suivez l'ordre des chapitres** - chaque section s'appuie sur la précédente
3. **Pratiquez chaque exemple** sur votre infrastructure
4. **Faites les exercices** à la fin de chaque section
5. **Consultez les références** pour approfondir

## 📚 Ressources additionnelles

- [Documentation officielle Kubernetes](https://kubernetes.io/docs/)
- [Kubeadm Documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
- [Kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

---

*Cette documentation est conçue pour vous accompagner de débutant à expert. Prenez votre temps et pratiquez régulièrement !* 