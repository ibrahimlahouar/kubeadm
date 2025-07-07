# 🚀 Installation de Kubernetes avec Kubeadm

## 📋 Prérequis

### Configuration minimale requise
- **Master Node** : 2 CPU, 2 GB RAM
- **Worker Node** : 1 CPU, 1 GB RAM
- **OS** : Ubuntu 20.04/22.04, CentOS 7/8, Debian 10/11

### Votre configuration (Excellente!)
- **VPS** : 12 cores, 48 GB RAM, 500 GB SSD
- **Recommandation** : Créez un cluster multi-nodes pour la pratique
  - 1 Master : 4 cores, 8 GB RAM
  - 3 Workers : 2-4 cores, 8-12 GB RAM chacun

## 🔧 Installation sur VPS

### Étape 1 : Préparation du système

```bash
# Sur TOUS les nodes (master et workers)

# 1. Mise à jour du système
sudo apt update && sudo apt upgrade -y

# 2. Désactiver le swap (requis par K8s)
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 3. Charger les modules kernel nécessaires
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 4. Configuration sysctl pour le réseau
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# 5. Vérifier les configurations
lsmod | grep br_netfilter
lsmod | grep overlay
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

### Étape 2 : Installation de containerd

```bash
# Installation de containerd (runtime recommandé)

# 1. Installer les dépendances
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release

# 2. Ajouter la clé GPG Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# 3. Ajouter le repository Docker
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 4. Installer containerd
sudo apt-get update
sudo apt-get install -y containerd.io

# 5. Configurer containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml

# 6. Modifier la configuration pour utiliser systemd comme cgroup driver
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# 7. Redémarrer containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### Étape 3 : Installation de kubeadm, kubelet et kubectl

```bash
# Sur TOUS les nodes

# 1. Installer les packages nécessaires
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

# 2. Ajouter la clé GPG de Google Cloud
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

# 3. Ajouter le repository Kubernetes
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# 4. Installer kubelet, kubeadm et kubectl
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# 5. Empêcher les mises à jour automatiques
sudo apt-mark hold kubelet kubeadm kubectl

# 6. Vérifier les versions
kubeadm version
kubelet --version
kubectl version --client
```

### Étape 4 : Initialisation du Master Node

```bash
# UNIQUEMENT sur le Master Node

# 1. Initialiser le cluster
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=<MASTER_IP>

# Remplacez <MASTER_IP> par l'IP de votre master node

# 2. Sauvegarder la sortie! Elle contient la commande join pour les workers
# Exemple de sortie :
# kubeadm join 192.168.1.100:6443 --token abcdef.0123456789abcdef \
#     --discovery-token-ca-cert-hash sha256:1234...

# 3. Configurer kubectl pour l'utilisateur actuel
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 4. Vérifier que le cluster fonctionne
kubectl get nodes
kubectl get pods -A
```

### Étape 5 : Installation du réseau (CNI)

```bash
# Sur le Master Node

# Option 1 : Flannel (simple et léger)
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Option 2 : Calico (plus de fonctionnalités)
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/custom-resources.yaml

# Option 3 : Weave Net
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml

# Attendre que tous les pods soient ready
watch kubectl get pods -A
```

### Étape 6 : Joindre les Worker Nodes

```bash
# Sur CHAQUE Worker Node

# Utiliser la commande fournie lors de l'init du master
sudo kubeadm join <MASTER_IP>:6443 --token <TOKEN> \
    --discovery-token-ca-cert-hash sha256:<HASH>

# Si vous avez perdu le token :
# Sur le master :
kubeadm token create --print-join-command
```

## 💻 Configuration de kubectl sur votre MacBook

### Option 1 : Copier le fichier kubeconfig

```bash
# Sur votre MacBook

# 1. Créer le répertoire .kube
mkdir -p ~/.kube

# 2. Copier le fichier config depuis le master
scp user@<MASTER_IP>:/home/user/.kube/config ~/.kube/config

# 3. Vérifier la connexion
kubectl get nodes
```

### Option 2 : Configuration manuelle

```bash
# Sur votre MacBook

# 1. Installer kubectl si ce n'est pas déjà fait
brew install kubectl

# 2. Créer la configuration
kubectl config set-cluster my-cluster --server=https://<MASTER_IP>:6443 --certificate-authority=ca.crt
kubectl config set-credentials admin --client-certificate=admin.crt --client-key=admin.key
kubectl config set-context my-context --cluster=my-cluster --user=admin
kubectl config use-context my-context
```

### Configuration SSH pour un accès facile

```bash
# ~/.ssh/config sur votre MacBook
Host k8s-master
    HostName <MASTER_IP>
    User ubuntu
    IdentityFile ~/.ssh/id_rsa
    
Host k8s-worker1
    HostName <WORKER1_IP>
    User ubuntu
    IdentityFile ~/.ssh/id_rsa
```

## 🔒 Sécurisation du cluster

### 1. Firewall (UFW)

```bash
# Sur le Master
sudo ufw allow 6443/tcp  # API server
sudo ufw allow 2379:2380/tcp  # etcd
sudo ufw allow 10250/tcp  # Kubelet API
sudo ufw allow 10251/tcp  # kube-scheduler
sudo ufw allow 10252/tcp  # kube-controller-manager

# Sur les Workers
sudo ufw allow 10250/tcp  # Kubelet API
sudo ufw allow 30000:32767/tcp  # NodePort Services

# Sur tous les nodes (pour le CNI)
sudo ufw allow from 10.244.0.0/16  # Pod network
```

### 2. Génération de certificats pour l'accès distant

```bash
# Sur le Master

# Créer un utilisateur pour l'accès distant
kubectl create serviceaccount remote-user -n kube-system
kubectl create clusterrolebinding remote-user-binding \
  --clusterrole=cluster-admin \
  --serviceaccount=kube-system:remote-user

# Récupérer le token
TOKEN=$(kubectl -n kube-system get secret $(kubectl -n kube-system get serviceaccount remote-user -o jsonpath='{.secrets[0].name}') -o jsonpath='{.data.token}' | base64 --decode)

echo $TOKEN
```

## 🧪 Tests de validation

### 1. Vérifier l'état du cluster

```bash
# État des nodes
kubectl get nodes -o wide

# Composants du système
kubectl get pods -n kube-system

# Version
kubectl version --short

# Informations du cluster
kubectl cluster-info
```

### 2. Déployer une application test

```bash
# Créer un déploiement nginx
kubectl create deployment nginx-test --image=nginx:latest --replicas=3

# Exposer le service
kubectl expose deployment nginx-test --port=80 --type=NodePort

# Vérifier
kubectl get all
```

### 3. Test de connectivité

```bash
# Depuis votre MacBook
curl http://<WORKER_IP>:<NODE_PORT>
```

## 🔧 Configuration avancée

### 1. Activation du dashboard Kubernetes

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

# Accès via proxy
kubectl proxy

# Accéder à : http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

### 2. Configuration du stockage

```bash
# Installer un StorageClass local
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
EOF
```

## 🚨 Dépannage courant

### Problèmes fréquents et solutions

```bash
# 1. Nodes en état NotReady
kubectl describe node <node-name>
# Vérifier : kubelet, réseau CNI, containerd

# 2. Pods en Pending
kubectl describe pod <pod-name>
# Vérifier : ressources disponibles, PVC, images

# 3. Réinitialiser un node
sudo kubeadm reset
sudo rm -rf /etc/cni/net.d
sudo rm -rf $HOME/.kube

# 4. Logs des composants
journalctl -u kubelet
journalctl -u containerd
kubectl logs -n kube-system <pod-name>
```

## 📚 Script d'installation automatisée

```bash
#!/bin/bash
# install-k8s.sh

set -e

echo "🚀 Installation de Kubernetes avec kubeadm"

# Variables
K8S_VERSION="1.28.0"
POD_NETWORK_CIDR="10.244.0.0/16"

# Fonction d'installation des prérequis
install_prerequisites() {
    echo "📦 Installation des prérequis..."
    sudo apt-get update
    sudo apt-get install -y apt-transport-https ca-certificates curl
    
    # Désactiver swap
    sudo swapoff -a
    sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
    
    # Modules kernel
    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
    
    sudo modprobe overlay
    sudo modprobe br_netfilter
    
    # Paramètres sysctl
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
    
    sudo sysctl --system
}

# Fonction d'installation de containerd
install_containerd() {
    echo "🐳 Installation de containerd..."
    # ... (code containerd)
}

# Fonction d'installation de kubeadm
install_kubeadm() {
    echo "☸️ Installation de kubeadm, kubelet et kubectl..."
    # ... (code kubeadm)
}

# Exécution
install_prerequisites
install_containerd
install_kubeadm

echo "✅ Installation terminée!"
```

## 🎯 Points clés à retenir

1. **Toujours sauvegarder** la commande join du master
2. **Vérifier les logs** en cas de problème
3. **Sécuriser l'accès** à l'API server
4. **Automatiser** avec des scripts pour la reproductibilité

---

[⬅️ Les Composants](./03-composants.md) | [🏠 Sommaire](./README.md) | [➡️ Les Objets Kubernetes](./05-objets-kubernetes.md) 