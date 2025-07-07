# 💾 Stockage Persistant dans Kubernetes

## 📍 Vue d'ensemble

Le stockage dans Kubernetes permet aux applications de persister des données au-delà du cycle de vie des pods. Kubernetes propose un système d'abstraction flexible pour gérer différents types de stockage.

## 🏗️ Architecture du stockage

```
┌─────────────────────────────────────────────────────┐
│              STORAGE ARCHITECTURE                    │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Application Pod                                    │
│       │                                            │
│       ▼                                            │
│  PersistentVolumeClaim (PVC)                       │
│       │                                            │
│       ▼                                            │
│  PersistentVolume (PV)                             │
│       │                                            │
│       ▼                                            │
│  StorageClass                                      │
│       │                                            │
│       ▼                                            │
│  Storage Backend (NFS, iSCSI, Cloud Disk, etc.)    │
│                                                     │
└─────────────────────────────────────────────────────┘
```

## 📦 Volumes éphémères

### emptyDir

Volume temporaire créé avec le pod et supprimé quand le pod est détruit.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-emptydir
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: cache-volume
      mountPath: /cache
  - name: sidecar
    image: busybox
    command: ['sh', '-c', 'while true; do echo $(date) >> /cache/log.txt; sleep 5; done']
    volumeMounts:
    - name: cache-volume
      mountPath: /cache
  volumes:
  - name: cache-volume
    emptyDir: {}
    # Optionnel : utiliser la RAM
    # emptyDir:
    #   medium: Memory
    #   sizeLimit: 1Gi
```

### hostPath

Monte un répertoire du node hôte dans le pod.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-hostpath
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: host-volume
      mountPath: /host-data
  volumes:
  - name: host-volume
    hostPath:
      path: /data
      type: DirectoryOrCreate  # Directory, File, Socket, etc.
```

⚠️ **Attention** : hostPath lie le pod à un node spécifique!

## 💽 PersistentVolume (PV) et PersistentVolumeClaim (PVC)

### Concept

- **PV** : Ressource de stockage dans le cluster
- **PVC** : Demande de stockage par un utilisateur

### Exemple de PersistentVolume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany  # RWO, ROX, RWX
  persistentVolumeReclaimPolicy: Retain  # Delete, Recycle
  storageClassName: nfs-storage
  nfs:
    server: nfs-server.example.com
    path: /exported/path
```

### Types d'accès (accessModes)

| Mode | Abréviation | Description |
|------|-------------|-------------|
| ReadWriteOnce | RWO | Lecture/écriture par un seul node |
| ReadOnlyMany | ROX | Lecture seule par plusieurs nodes |
| ReadWriteMany | RWX | Lecture/écriture par plusieurs nodes |

### Exemple de PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-app
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 5Gi
  # Optionnel : sélecteur de PV
  selector:
    matchLabels:
      environment: production
```

### Utilisation dans un Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-pvc
spec:
  containers:
  - name: app
    image: mysql:5.7
    volumeMounts:
    - name: mysql-storage
      mountPath: /var/lib/mysql
  volumes:
  - name: mysql-storage
    persistentVolumeClaim:
      claimName: pvc-app
```

## 🎯 StorageClass

### Qu'est-ce qu'une StorageClass ?

Une StorageClass permet le provisionnement dynamique de volumes.

### Exemple avec différents providers

```yaml
# 1. StorageClass pour AWS EBS
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iopsPerGB: "10"
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

---
# 2. StorageClass pour Local Storage
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer

---
# 3. StorageClass par défaut
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
```

### Provisionnement dynamique

```yaml
# PVC qui déclenche la création automatique d'un PV
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd  # StorageClass existante
  resources:
    requests:
      storage: 100Gi
```

## 🗄️ StatefulSet et stockage

StatefulSet garantit un stockage stable pour chaque réplique.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-statefulset
spec:
  serviceName: postgres-service
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:13
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: mydb
        - name: POSTGRES_USER
          value: admin
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
  # Template de PVC pour chaque pod
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 50Gi
```

## 🔌 CSI (Container Storage Interface)

### Architecture CSI

```
┌─────────────────────────────────────────────────┐
│                CSI ARCHITECTURE                  │
├─────────────────────────────────────────────────┤
│                                                 │
│  Kubernetes Control Plane                       │
│       │                                        │
│       ▼                                        │
│  CSI Controller Plugin                         │
│       │                                        │
│       ├── CreateVolume()                       │
│       ├── DeleteVolume()                       │
│       └── ControllerPublishVolume()            │
│                                                 │
│  Kubernetes Node                               │
│       │                                        │
│       ▼                                        │
│  CSI Node Plugin                               │
│       │                                        │
│       ├── NodeStageVolume()                    │
│       ├── NodePublishVolume()                  │
│       └── NodeUnpublishVolume()                │
│                                                 │
└─────────────────────────────────────────────────┘
```

### Installation d'un driver CSI (exemple avec NFS)

```bash
# Installation du driver NFS CSI
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs --namespace kube-system

# StorageClass pour NFS CSI
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: nfs-server.example.com
  share: /exported/dynamic
reclaimPolicy: Delete
volumeBindingMode: Immediate
EOF
```

## 🚀 Patterns de stockage avancés

### 1. Volume Snapshots

```yaml
# VolumeSnapshotClass
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-snapclass
driver: pd.csi.storage.gke.io
deletionPolicy: Delete

---
# Créer un snapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-snapshot
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: postgres-pvc

---
# Restaurer depuis un snapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-restore
spec:
  dataSource:
    name: postgres-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
```

### 2. Volume Cloning

```yaml
# Clone d'un PVC existant
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cloned-pvc
spec:
  dataSource:
    name: source-pvc
    kind: PersistentVolumeClaim
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

### 3. Expansion de volume

```yaml
# Activer l'expansion dans StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable-storage
provisioner: kubernetes.io/aws-ebs
allowVolumeExpansion: true

---
# Augmenter la taille d'un PVC
kubectl patch pvc my-pvc -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'
```

## 📊 Exemples pratiques

### 1. Application WordPress avec MySQL

```yaml
# MySQL StatefulSet avec stockage persistant
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        - name: MYSQL_DATABASE
          value: wordpress
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-persistent-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 20Gi

---
# WordPress Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
spec:
  replicas: 2
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: wordpress:latest
        ports:
        - containerPort: 80
        env:
        - name: WORDPRESS_DB_HOST
          value: mysql
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-persistent-storage
        persistentVolumeClaim:
          claimName: wordpress-pvc
```

### 2. Backup automatisé avec CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
spec:
  schedule: "0 2 * * *"  # 2h du matin chaque jour
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: postgres-backup
            image: postgres:13
            command:
            - /bin/bash
            - -c
            - |
              DATE=$(date +%Y%m%d_%H%M%S)
              pg_dump -h postgres-service -U admin mydb > /backup/backup_$DATE.sql
              # Garder seulement les 7 derniers backups
              ls -t /backup/backup_*.sql | tail -n +8 | xargs rm -f
            env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            volumeMounts:
            - name: backup-volume
              mountPath: /backup
          restartPolicy: OnFailure
          volumes:
          - name: backup-volume
            persistentVolumeClaim:
              claimName: backup-pvc
```

## 🛠️ Dépannage du stockage

### Commandes utiles

```bash
# Vérifier l'état des PV/PVC
kubectl get pv
kubectl get pvc -A

# Décrire un PVC pour voir les events
kubectl describe pvc my-pvc

# Vérifier l'utilisation du stockage dans un pod
kubectl exec -it my-pod -- df -h

# Logs du provisioner
kubectl logs -n kube-system deployment/csi-provisioner

# Vérifier les StorageClasses
kubectl get storageclass

# Voir les VolumeAttachments (CSI)
kubectl get volumeattachments
```

### Problèmes courants

| Problème | Cause possible | Solution |
|----------|----------------|----------|
| PVC en Pending | Pas de PV disponible | Vérifier StorageClass et provisioner |
| Mount failed | Node sans accès au storage | Vérifier la connectivité réseau |
| PVC bound mais pod en erreur | Permissions ou format | Vérifier les logs du pod |
| Volume plein | Espace insuffisant | Expansion ou nettoyage |

## 🎯 Bonnes pratiques

1. **Utilisez des StorageClasses** pour le provisionnement dynamique
2. **Définissez des limites** de stockage appropriées
3. **Sauvegardez régulièrement** vos données importantes
4. **Testez la restauration** de vos backups
5. **Utilisez StatefulSet** pour les applications avec état
6. **Monitoring** de l'utilisation du stockage
7. **ReclaimPolicy** : Utilisez "Retain" pour les données critiques

## 📝 Exercices pratiques

1. Déployez une base de données PostgreSQL avec stockage persistant
2. Configurez des snapshots automatiques
3. Testez l'expansion d'un volume
4. Migrez des données entre deux PVC
5. Implémentez une stratégie de backup/restore

---

[⬅️ Networking](./06-networking.md) | [🏠 Sommaire](./README.md) | [➡️ Sécurité](./08-security.md) 