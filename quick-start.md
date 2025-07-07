# 🚀 Guide de Démarrage Rapide Kubernetes

## 📋 Checklist d'installation sur votre VPS

```bash
# 1. Préparer le système (sur tous les nodes)
sudo apt update && sudo apt upgrade -y
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 2. Installer containerd
sudo apt install -y containerd.io
sudo systemctl enable containerd
sudo systemctl start containerd

# 3. Installer kubeadm, kubelet, kubectl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# 4. Initialiser le master
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# 5. Configurer kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 6. Installer le réseau (Flannel)
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# 7. Permettre le scheduling sur le master (optionnel pour test)
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

## 🛠️ Commandes Kubectl Essentielles

### Gestion des ressources
```bash
# Créer des ressources
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort

# Appliquer des configurations
kubectl apply -f deployment.yaml
kubectl apply -f ./k8s/

# Supprimer des ressources
kubectl delete deployment nginx
kubectl delete -f deployment.yaml

# Mise à jour
kubectl set image deployment/nginx nginx=nginx:1.19
kubectl scale deployment nginx --replicas=5
kubectl autoscale deployment nginx --min=2 --max=10 --cpu-percent=80
```

### Inspection et débogage
```bash
# Lister les ressources
kubectl get pods -A                    # Tous les pods de tous les namespaces
kubectl get all -n kube-system        # Toutes les ressources d'un namespace
kubectl get nodes -o wide             # Nodes avec infos détaillées

# Décrire les ressources
kubectl describe pod nginx-xxx
kubectl describe node worker-1

# Logs
kubectl logs nginx-xxx
kubectl logs -f nginx-xxx             # Follow
kubectl logs nginx-xxx -c container   # Container spécifique
kubectl logs --previous nginx-xxx     # Logs du container précédent

# Exec dans un pod
kubectl exec -it nginx-xxx -- bash
kubectl exec nginx-xxx -- ls /app

# Port forwarding
kubectl port-forward pod/nginx-xxx 8080:80
kubectl port-forward svc/nginx-service 8080:80

# Copier des fichiers
kubectl cp nginx-xxx:/app/config.yaml ./config.yaml
kubectl cp ./data.txt nginx-xxx:/tmp/
```

### Gestion des contextes
```bash
# Voir les contextes
kubectl config get-contexts
kubectl config current-context

# Changer de contexte
kubectl config use-context production

# Créer un namespace et l'utiliser
kubectl create namespace dev
kubectl config set-context --current --namespace=dev
```

### Alias utiles (.bashrc ou .zshrc)
```bash
# Alias kubectl
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deployment'
alias kaf='kubectl apply -f'
alias kdel='kubectl delete'
alias kdes='kubectl describe'
alias klog='kubectl logs'
alias kexec='kubectl exec -it'

# Completion
source <(kubectl completion bash)  # ou zsh

# Fonction pour changer rapidement de namespace
kns() {
  kubectl config set-context --current --namespace=$1
}
```

## 🐳 Connexion depuis votre MacBook

### Configuration kubectl
```bash
# Sur votre MacBook
# 1. Installer kubectl
brew install kubectl

# 2. Copier la config depuis le master
scp user@<VPS_IP>:~/.kube/config ~/.kube/config

# 3. Tester la connexion
kubectl get nodes

# 4. Configuration SSH pour simplifier
cat >> ~/.ssh/config << EOF
Host k8s-master
    HostName <VPS_IP>
    User ubuntu
    IdentityFile ~/.ssh/id_rsa
    LocalForward 6443 localhost:6443
EOF

# 5. Tunnel SSH si nécessaire
ssh -L 6443:localhost:6443 k8s-master
```

### Docker Desktop + Kubernetes
```bash
# Activer Kubernetes dans Docker Desktop
# Preferences > Kubernetes > Enable Kubernetes

# Basculer entre les contextes
kubectl config use-context docker-desktop  # Local
kubectl config use-context k8s-vps        # VPS
```

## 📦 Déploiement rapide d'une app

### 1. Application simple (nginx)
```yaml
# app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

```bash
# Déployer
kubectl apply -f app.yaml

# Vérifier
kubectl get all
curl http://<NODE_IP>:30080
```

### 2. Application avec base de données
```bash
# Utiliser Helm pour PostgreSQL
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-postgres bitnami/postgresql

# Récupérer le mot de passe
export POSTGRES_PASSWORD=$(kubectl get secret --namespace default my-postgres-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)

# Se connecter
kubectl run my-postgres-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:14 --env="PGPASSWORD=$POSTGRES_PASSWORD" --command -- psql --host my-postgres-postgresql -U postgres
```

## 🔧 Outils recommandés

### k9s - Terminal UI
```bash
# Installation
brew install k9s  # MacOS
snap install k9s  # Linux

# Utilisation
k9s
```

### Lens - IDE Kubernetes
```bash
# Télécharger depuis https://k8slens.dev/
# Ajouter votre cluster via kubeconfig
```

### kubectx/kubens
```bash
# Installation
brew install kubectx

# Utilisation
kubectx          # Liste les contextes
kubectx prod     # Change de contexte
kubens           # Liste les namespaces
kubens dev       # Change de namespace
```

### stern - Logs multi-pods
```bash
# Installation
brew install stern

# Utilisation
stern my-app     # Logs de tous les pods my-app
stern . -n dev   # Tous les logs du namespace dev
```

## 📊 Dashboard Kubernetes

```bash
# Installer le dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

# Créer un compte admin
kubectl create serviceaccount dashboard-admin -n kubernetes-dashboard
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:dashboard-admin

# Obtenir le token
kubectl -n kubernetes-dashboard create token dashboard-admin

# Accès via proxy
kubectl proxy
# Ouvrir: http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

## 🚨 Commandes de dépannage

```bash
# Pod en erreur
kubectl describe pod <pod-name>
kubectl logs <pod-name> --previous
kubectl get events --sort-by='.lastTimestamp'

# Node en problème
kubectl describe node <node-name>
kubectl top nodes
kubectl cordon <node-name>    # Empêche nouveaux pods
kubectl drain <node-name>     # Évacue les pods

# Ressources
kubectl top pods --all-namespaces
kubectl describe quota -A
kubectl describe limits -A

# Réseau
kubectl run test --image=busybox -it --rm -- sh
# Dans le pod:
nslookup kubernetes.default
wget -O- http://service-name

# Certificats
kubeadm certs check-expiration
kubeadm certs renew all
```

## 📚 Prochaines étapes

1. **Semaine 1** : Parcourir les chapitres 1-3 (Introduction, Architecture, Composants)
2. **Semaine 2** : Installation sur votre VPS (chapitre 4) et premiers déploiements
3. **Semaine 3** : Maîtriser les objets K8s (chapitres 5-6)
4. **Semaine 4** : Storage et sécurité (chapitres 7-8)
5. **Semaine 5** : Monitoring et projets pratiques (chapitres 9-10)

## 🎯 Tips pour devenir expert

1. **Pratiquez quotidiennement** : 30 min/jour minimum
2. **Cassez des choses** : N'ayez pas peur d'expérimenter
3. **Lisez les logs** : Comprenez les erreurs
4. **Automatisez** : Scripts, Helm, GitOps
5. **Contribuez** : Participez à des projets open source
6. **Certifiez-vous** : CKA, CKAD, CKS

---

🎉 **Bon apprentissage !** N'hésitez pas à revenir sur ce guide régulièrement. 