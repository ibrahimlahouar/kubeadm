# 💪 Exercices Pratiques - Projets Kubernetes

## 📍 Vue d'ensemble

Ce chapitre contient des projets pratiques pour mettre en œuvre tout ce que vous avez appris. Chaque projet augmente en complexité.

## 🚀 Projet 1 : Application Todo List

### Objectif
Déployer une application Todo List avec frontend React, backend Node.js et base de données MongoDB.

### Architecture
```
┌─────────────────────────────────────────────────┐
│                 TODO APP                         │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐ │
│  │ Frontend │───▶│ Backend  │───▶│ MongoDB  │ │
│  │  React   │    │ Node.js  │    │          │ │
│  └──────────┘    └──────────┘    └──────────┘ │
│                                                 │
└─────────────────────────────────────────────────┘
```

### Étape 1 : Namespace et ConfigMap

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: todo-app

---
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: todo-config
  namespace: todo-app
data:
  API_URL: "http://backend-service:3000"
  DB_NAME: "tododb"
```

### Étape 2 : MongoDB StatefulSet

```yaml
# mongodb.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-secret
  namespace: todo-app
type: Opaque
data:
  mongodb-root-password: YWRtaW4xMjM=  # admin123

---
apiVersion: v1
kind: Service
metadata:
  name: mongodb
  namespace: todo-app
spec:
  clusterIP: None
  selector:
    app: mongodb
  ports:
  - port: 27017
    targetPort: 27017

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
  namespace: todo-app
spec:
  serviceName: mongodb
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:5.0
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          value: admin
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongodb-root-password
        - name: MONGO_INITDB_DATABASE
          valueFrom:
            configMapKeyRef:
              name: todo-config
              key: DB_NAME
        volumeMounts:
        - name: mongodb-storage
          mountPath: /data/db
  volumeClaimTemplates:
  - metadata:
      name: mongodb-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 5Gi
```

### Étape 3 : Backend Deployment

```yaml
# backend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: todo-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: node:16-alpine
        workingDir: /app
        command: ["sh", "-c"]
        args:
        - |
          cat > package.json << EOF
          {
            "name": "todo-backend",
            "version": "1.0.0",
            "main": "server.js",
            "dependencies": {
              "express": "^4.18.0",
              "mongoose": "^6.0.0",
              "cors": "^2.8.5"
            }
          }
          EOF
          
          cat > server.js << 'EOF'
          const express = require('express');
          const mongoose = require('mongoose');
          const cors = require('cors');
          
          const app = express();
          app.use(cors());
          app.use(express.json());
          
          // MongoDB connection
          const mongoUrl = `mongodb://admin:${process.env.MONGO_PASSWORD}@mongodb:27017/${process.env.DB_NAME}?authSource=admin`;
          mongoose.connect(mongoUrl);
          
          // Todo model
          const Todo = mongoose.model('Todo', {
            text: String,
            completed: Boolean
          });
          
          // Routes
          app.get('/api/todos', async (req, res) => {
            const todos = await Todo.find();
            res.json(todos);
          });
          
          app.post('/api/todos', async (req, res) => {
            const todo = new Todo({
              text: req.body.text,
              completed: false
            });
            await todo.save();
            res.json(todo);
          });
          
          app.put('/api/todos/:id', async (req, res) => {
            const todo = await Todo.findByIdAndUpdate(
              req.params.id,
              req.body,
              { new: true }
            );
            res.json(todo);
          });
          
          app.delete('/api/todos/:id', async (req, res) => {
            await Todo.findByIdAndDelete(req.params.id);
            res.json({ message: 'Deleted' });
          });
          
          app.listen(3000, () => {
            console.log('Backend running on port 3000');
          });
          EOF
          
          npm install
          node server.js
        ports:
        - containerPort: 3000
        env:
        - name: DB_NAME
          valueFrom:
            configMapKeyRef:
              name: todo-config
              key: DB_NAME
        - name: MONGO_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongodb-root-password
        livenessProbe:
          httpGet:
            path: /api/todos
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /api/todos
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: todo-app
spec:
  selector:
    app: backend
  ports:
  - port: 3000
    targetPort: 3000
```

### Étape 4 : Frontend Deployment

```yaml
# frontend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: todo-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      initContainers:
      - name: setup-html
        image: busybox
        command: ['sh', '-c']
        args:
        - |
          cat > /html/index.html << 'EOF'
          <!DOCTYPE html>
          <html>
          <head>
              <title>Todo App</title>
              <style>
                  body { font-family: Arial, sans-serif; max-width: 600px; margin: 50px auto; }
                  h1 { color: #333; }
                  input { padding: 10px; width: 70%; }
                  button { padding: 10px 20px; background: #007bff; color: white; border: none; cursor: pointer; }
                  ul { list-style: none; padding: 0; }
                  li { padding: 10px; margin: 5px 0; background: #f0f0f0; display: flex; justify-content: space-between; }
                  .completed { text-decoration: line-through; opacity: 0.6; }
              </style>
          </head>
          <body>
              <h1>📝 Todo List Kubernetes</h1>
              <div>
                  <input type="text" id="todoInput" placeholder="Ajouter une tâche...">
                  <button onclick="addTodo()">Ajouter</button>
              </div>
              <ul id="todoList"></ul>
              
              <script>
                  const API_URL = '/api';
                  
                  async function fetchTodos() {
                      const response = await fetch(`${API_URL}/todos`);
                      const todos = await response.json();
                      const list = document.getElementById('todoList');
                      list.innerHTML = '';
                      
                      todos.forEach(todo => {
                          const li = document.createElement('li');
                          li.className = todo.completed ? 'completed' : '';
                          li.innerHTML = `
                              <span onclick="toggleTodo('${todo._id}')">${todo.text}</span>
                              <button onclick="deleteTodo('${todo._id}')">Supprimer</button>
                          `;
                          list.appendChild(li);
                      });
                  }
                  
                  async function addTodo() {
                      const input = document.getElementById('todoInput');
                      if (input.value.trim()) {
                          await fetch(`${API_URL}/todos`, {
                              method: 'POST',
                              headers: { 'Content-Type': 'application/json' },
                              body: JSON.stringify({ text: input.value })
                          });
                          input.value = '';
                          fetchTodos();
                      }
                  }
                  
                  async function toggleTodo(id) {
                      await fetch(`${API_URL}/todos/${id}`, {
                          method: 'PUT',
                          headers: { 'Content-Type': 'application/json' },
                          body: JSON.stringify({ completed: true })
                      });
                      fetchTodos();
                  }
                  
                  async function deleteTodo(id) {
                      await fetch(`${API_URL}/todos/${id}`, { method: 'DELETE' });
                      fetchTodos();
                  }
                  
                  // Initial load
                  fetchTodos();
              </script>
          </body>
          </html>
          EOF
          
          cat > /html/nginx.conf << 'EOF'
          server {
              listen 80;
              location / {
                  root /usr/share/nginx/html;
                  index index.html;
              }
              location /api/ {
                  proxy_pass http://backend-service:3000/api/;
              }
          }
          EOF
        volumeMounts:
        - name: html
          mountPath: /html
      volumes:
      - name: html
        emptyDir: {}

---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: todo-app
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

### Étape 5 : Ingress et NetworkPolicy

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: todo-ingress
  namespace: todo-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: todo.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80

---
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: todo-netpol
  namespace: todo-app
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector: {}
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
  egress:
  - to:
    - podSelector: {}
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

### Déploiement

```bash
# Déployer l'application
kubectl apply -f namespace.yaml
kubectl apply -f configmap.yaml
kubectl apply -f mongodb.yaml
kubectl apply -f backend.yaml
kubectl apply -f frontend.yaml
kubectl apply -f ingress.yaml
kubectl apply -f network-policy.yaml

# Vérifier le déploiement
kubectl get all -n todo-app

# Accéder à l'application
# NodePort: http://<node-ip>:30080
# Ou configurer /etc/hosts avec l'IP et accéder via http://todo.example.com
```

## 🛍️ Projet 2 : E-commerce Microservices

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  E-COMMERCE PLATFORM                     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌──────────┐ │
│  │Frontend │  │Products │  │  Cart   │  │  Orders  │ │
│  │  SPA    │  │   API   │  │   API   │  │   API    │ │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬─────┘ │
│       │            │            │            │         │
│       └────────────┴────────────┴────────────┘        │
│                         │                              │
│                    ┌────┴────┐                         │
│                    │ Gateway │                         │
│                    └─────────┘                         │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│  │PostgreSQL│  │  Redis   │  │ RabbitMQ │            │
│  └──────────┘  └──────────┘  └──────────┘            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Structure des microservices

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ecommerce
  labels:
    istio-injection: enabled  # Pour service mesh

---
# common-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: common-config
  namespace: ecommerce
data:
  POSTGRES_HOST: "postgres-service"
  REDIS_HOST: "redis-service"
  RABBITMQ_HOST: "rabbitmq-service"
```

### Base de données PostgreSQL

```yaml
# postgres.yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: ecommerce
type: Opaque
data:
  postgres-password: cG9zdGdyZXMxMjM=  # postgres123

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: ecommerce
spec:
  serviceName: postgres-service
  replicas: 1
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
        image: postgres:14
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: postgres-password
        - name: POSTGRES_DB
          value: ecommerce
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi

---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: ecommerce
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

### Products Microservice

```yaml
# products-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: products-api
  namespace: ecommerce
spec:
  replicas: 3
  selector:
    matchLabels:
      app: products-api
  template:
    metadata:
      labels:
        app: products-api
        version: v1
    spec:
      containers:
      - name: products
        image: node:16-alpine
        workingDir: /app
        command: ["sh", "-c"]
        args:
        - |
          # Installation des dépendances
          npm init -y
          npm install express pg body-parser
          
          # Code du service
          cat > server.js << 'EOF'
          const express = require('express');
          const { Pool } = require('pg');
          const bodyParser = require('body-parser');
          
          const app = express();
          app.use(bodyParser.json());
          
          const pool = new Pool({
            host: process.env.POSTGRES_HOST,
            database: 'ecommerce',
            user: 'postgres',
            password: process.env.POSTGRES_PASSWORD,
            port: 5432,
          });
          
          // Initialisation de la base
          async function initDB() {
            try {
              await pool.query(`
                CREATE TABLE IF NOT EXISTS products (
                  id SERIAL PRIMARY KEY,
                  name VARCHAR(255) NOT NULL,
                  description TEXT,
                  price DECIMAL(10, 2) NOT NULL,
                  stock INTEGER NOT NULL
                )
              `);
              
              // Données de test
              const count = await pool.query('SELECT COUNT(*) FROM products');
              if (count.rows[0].count === '0') {
                await pool.query(`
                  INSERT INTO products (name, description, price, stock) VALUES
                  ('Laptop', 'High-performance laptop', 999.99, 50),
                  ('Mouse', 'Wireless mouse', 29.99, 200),
                  ('Keyboard', 'Mechanical keyboard', 79.99, 150)
                `);
              }
            } catch (err) {
              console.error('DB Init Error:', err);
            }
          }
          
          initDB();
          
          // Routes
          app.get('/health', (req, res) => res.json({ status: 'ok' }));
          
          app.get('/api/products', async (req, res) => {
            try {
              const result = await pool.query('SELECT * FROM products');
              res.json(result.rows);
            } catch (err) {
              res.status(500).json({ error: err.message });
            }
          });
          
          app.get('/api/products/:id', async (req, res) => {
            try {
              const result = await pool.query('SELECT * FROM products WHERE id = $1', [req.params.id]);
              if (result.rows.length === 0) {
                return res.status(404).json({ error: 'Product not found' });
              }
              res.json(result.rows[0]);
            } catch (err) {
              res.status(500).json({ error: err.message });
            }
          });
          
          app.post('/api/products', async (req, res) => {
            const { name, description, price, stock } = req.body;
            try {
              const result = await pool.query(
                'INSERT INTO products (name, description, price, stock) VALUES ($1, $2, $3, $4) RETURNING *',
                [name, description, price, stock]
              );
              res.status(201).json(result.rows[0]);
            } catch (err) {
              res.status(500).json({ error: err.message });
            }
          });
          
          const PORT = process.env.PORT || 3001;
          app.listen(PORT, () => {
            console.log(`Products service running on port ${PORT}`);
          });
          EOF
          
          node server.js
        ports:
        - containerPort: 3001
        env:
        - name: PORT
          value: "3001"
        - name: POSTGRES_HOST
          valueFrom:
            configMapKeyRef:
              name: common-config
              key: POSTGRES_HOST
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: postgres-password
        livenessProbe:
          httpGet:
            path: /health
            port: 3001
          initialDelaySeconds: 30
        readinessProbe:
          httpGet:
            path: /health
            port: 3001
          initialDelaySeconds: 10
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"

---
apiVersion: v1
kind: Service
metadata:
  name: products-service
  namespace: ecommerce
  labels:
    app: products-api
spec:
  selector:
    app: products-api
  ports:
  - port: 3001
    targetPort: 3001
    name: http
```

### API Gateway

```yaml
# api-gateway.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  namespace: ecommerce
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
      - name: gateway
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: config
          mountPath: /etc/nginx/conf.d
      volumes:
      - name: config
        configMap:
          name: gateway-config

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: gateway-config
  namespace: ecommerce
data:
  default.conf: |
    upstream products {
        server products-service:3001;
    }
    
    upstream cart {
        server cart-service:3002;
    }
    
    upstream orders {
        server orders-service:3003;
    }
    
    server {
        listen 80;
        
        location /api/products {
            proxy_pass http://products;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
        
        location /api/cart {
            proxy_pass http://cart;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
        
        location /api/orders {
            proxy_pass http://orders;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
        
        location /health {
            return 200 "OK\n";
        }
    }

---
apiVersion: v1
kind: Service
metadata:
  name: gateway-service
  namespace: ecommerce
spec:
  type: LoadBalancer
  selector:
    app: api-gateway
  ports:
  - port: 80
    targetPort: 80
```

### Monitoring et Observabilité

```yaml
# monitoring.yaml
apiVersion: v1
kind: ServiceMonitor
metadata:
  name: ecommerce-metrics
  namespace: ecommerce
spec:
  selector:
    matchLabels:
      monitored: "true"
  endpoints:
  - port: metrics
    interval: 30s

---
# Grafana Dashboard ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard
  namespace: ecommerce
data:
  dashboard.json: |
    {
      "dashboard": {
        "title": "E-commerce Microservices",
        "panels": [
          {
            "title": "Request Rate",
            "targets": [{
              "expr": "sum(rate(http_requests_total[5m])) by (service)"
            }]
          },
          {
            "title": "Error Rate",
            "targets": [{
              "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) by (service)"
            }]
          },
          {
            "title": "Response Time",
            "targets": [{
              "expr": "histogram_quantile(0.95, http_request_duration_seconds_bucket)"
            }]
          }
        ]
      }
    }
```

### Déploiement avec Helm

```bash
# helm-chart/Chart.yaml
apiVersion: v2
name: ecommerce-platform
description: E-commerce microservices platform
version: 1.0.0

# helm-chart/values.yaml
replicaCount:
  frontend: 3
  products: 3
  cart: 2
  orders: 2

image:
  tag: latest
  pullPolicy: IfNotPresent

ingress:
  enabled: true
  host: shop.example.com

postgresql:
  enabled: true
  auth:
    postgresPassword: postgres123

redis:
  enabled: true

monitoring:
  enabled: true
```

## 🎮 Projet 3 : CI/CD Pipeline avec GitOps

### Architecture GitOps

```
┌─────────────────────────────────────────────────┐
│                GitOps Pipeline                   │
├─────────────────────────────────────────────────┤
│                                                 │
│  Developer ──push──► Git Repo                  │
│                         │                       │
│                         ▼                       │
│                    CI Pipeline                  │
│                   (Jenkins/GitLab)              │
│                         │                       │
│                         ▼                       │
│                  Build & Test                   │
│                         │                       │
│                         ▼                       │
│                  Push to Registry               │
│                         │                       │
│                         ▼                       │
│                 Update Manifests                │
│                         │                       │
│                         ▼                       │
│                    ArgoCD                       │
│                         │                       │
│                         ▼                       │
│                 Deploy to K8s                   │
│                                                 │
└─────────────────────────────────────────────────┘
```

### Jenkins sur Kubernetes

```yaml
# jenkins.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: jenkins

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: jenkins

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: jenkins
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["*"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["*"]
- apiGroups: ["batch"]
  resources: ["jobs"]
  verbs: ["*"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins
subjects:
- kind: ServiceAccount
  name: jenkins
  namespace: jenkins

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: jenkins
  namespace: jenkins
spec:
  serviceName: jenkins
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      serviceAccountName: jenkins
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts
        ports:
        - containerPort: 8080
        - containerPort: 50000
        env:
        - name: JAVA_OPTS
          value: "-Djenkins.install.runSetupWizard=false"
        volumeMounts:
        - name: jenkins-home
          mountPath: /var/jenkins_home
        - name: docker-sock
          mountPath: /var/run/docker.sock
      volumes:
      - name: docker-sock
        hostPath:
          path: /var/run/docker.sock
  volumeClaimTemplates:
  - metadata:
      name: jenkins-home
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 20Gi
```

### Pipeline Jenkins (Jenkinsfile)

```groovy
// Jenkinsfile
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: docker
    image: docker:dind
    tty: true
    securityContext:
      privileged: true
  - name: kubectl
    image: bitnami/kubectl:latest
    tty: true
    command:
    - cat
  - name: helm
    image: alpine/helm:latest
    tty: true
    command:
    - cat
"""
        }
    }
    
    environment {
        DOCKER_REGISTRY = "registry.example.com"
        APP_NAME = "myapp"
        GIT_REPO = "https://github.com/user/k8s-manifests.git"
    }
    
    stages {
        stage('Build') {
            steps {
                container('docker') {
                    sh """
                        docker build -t ${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER} .
                        docker tag ${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER} ${DOCKER_REGISTRY}/${APP_NAME}:latest
                    """
                }
            }
        }
        
        stage('Test') {
            steps {
                container('docker') {
                    sh """
                        docker run --rm ${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER} npm test
                    """
                }
            }
        }
        
        stage('Push') {
            steps {
                container('docker') {
                    withCredentials([usernamePassword(credentialsId: 'docker-registry', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                        sh """
                            docker login -u $USER -p $PASS ${DOCKER_REGISTRY}
                            docker push ${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER}
                            docker push ${DOCKER_REGISTRY}/${APP_NAME}:latest
                        """
                    }
                }
            }
        }
        
        stage('Update Manifests') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github', usernameVariable: 'USER', passwordVariable: 'TOKEN')]) {
                    sh """
                        git clone https://$USER:$TOKEN@github.com/user/k8s-manifests.git
                        cd k8s-manifests
                        sed -i 's|image: .*|image: ${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER}|' deployment.yaml
                        git config user.email "jenkins@example.com"
                        git config user.name "Jenkins"
                        git add deployment.yaml
                        git commit -m "Update image to ${BUILD_NUMBER}"
                        git push origin main
                    """
                }
            }
        }
    }
}
```

### ArgoCD Installation et Configuration

```yaml
# argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/user/k8s-manifests
    targetRevision: HEAD
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - Validate=true
    - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### Kustomize pour la gestion des environnements

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- configmap.yaml

# overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
- ../../base

patchesStrategicMerge:
- deployment-patch.yaml

configMapGenerator:
- name: app-config
  behavior: merge
  literals:
  - ENVIRONMENT=development
  - LOG_LEVEL=debug

# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
- ../../base

replicas:
- name: myapp
  count: 5

images:
- name: myapp
  newTag: stable

resources:
- hpa.yaml
- pdb.yaml

configMapGenerator:
- name: app-config
  behavior: merge
  literals:
  - ENVIRONMENT=production
  - LOG_LEVEL=info
```

## 🎯 Commandes utiles pour les projets

```bash
# Debugging
kubectl logs -f deployment/myapp -n production
kubectl exec -it deployment/myapp -n production -- sh
kubectl describe pod -n production
kubectl get events -n production --sort-by='.lastTimestamp'

# Monitoring
kubectl top nodes
kubectl top pods -n production
watch kubectl get hpa -n production

# Port forwarding pour tests locaux
kubectl port-forward svc/frontend-service 8080:80 -n todo-app

# Dry run pour valider
kubectl apply --dry-run=client -f deployment.yaml
kubectl diff -f deployment.yaml

# Rollback
kubectl rollout undo deployment/myapp -n production
kubectl rollout history deployment/myapp -n production
```

## 📚 Ressources pour aller plus loin

1. **Patterns avancés**
   - Sidecar containers
   - Init containers
   - Pod disruption budgets
   - Priority classes

2. **Outils complémentaires**
   - Tekton pour CI/CD natif K8s
   - Flux pour GitOps
   - Crossplane pour infrastructure as code
   - Open Policy Agent pour policies

3. **Certifications recommandées**
   - CKA (Certified Kubernetes Administrator)
   - CKAD (Certified Kubernetes Application Developer)
   - CKS (Certified Kubernetes Security Specialist)

---

[⬅️ Monitoring](./09-monitoring.md) | [🏠 Retour au sommaire](./README.md)

🎉 **Félicitations !** Vous avez maintenant toutes les clés pour maîtriser Kubernetes. Continuez à pratiquer et à explorer ! 