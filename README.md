# Helm — MongoDB ReplicaSet on Linode Kubernetes Engine (LKE)

> **FR** — Déploiement de MongoDB en mode ReplicaSet avec persistance sur des volumes Linode Block Storage, d'un client Mongo Express, et configuration d'un Nginx Ingress Controller qui provisionne automatiquement un NodeBalancer cloud pour exposer l'application.
>
> **EN** — Deployment of MongoDB as a ReplicaSet with persistence on Linode Block Storage volumes, a Mongo Express UI client, and configuration of an Nginx Ingress Controller that automatically provisions a cloud NodeBalancer to expose the application.

---

## Stack

![Helm](https://img.shields.io/badge/Helm-3-0F1689?logo=helm)
![Kubernetes](https://img.shields.io/badge/Kubernetes-LKE-326CE5?logo=kubernetes)
![MongoDB](https://img.shields.io/badge/MongoDB-ReplicaSet-47A248?logo=mongodb)
![Nginx](https://img.shields.io/badge/Nginx-Ingress-009639?logo=nginx)
![Linode](https://img.shields.io/badge/Linode-Akamai-00A95C?logo=linode)

---

## FR — Description

Ce projet déploie une stack MongoDB + Mongo Express sur un cluster Kubernetes managé Linode (LKE) en exploitant Helm pour orchestrer les ressources complexes.

### Étape 1 — Cluster LKE

Provisionnement d'un cluster Kubernetes managé sur Linode (2 nœuds, plan 4 GB). Configuration de `kubectl` via le fichier `kubeconfig.yaml` téléchargé depuis la console Linode.

### Étape 2 — MongoDB via Helm (Bitnami)

**Ressources déployées :** `StatefulSet` (3 réplicas + 1 arbiter), `Service` headless, `Secret`, `PersistentVolumeClaim` (un par pod)

**Concepts démontrés :**
- Surcharge des valeurs par défaut d'un chart Helm via un fichier `values.yaml` personnalisé
- Mode `replicaset` : 3 instances de données + 1 arbiter pour les élections de primaire
- `storageClass: linode-block-storage` : provisionnement dynamique de volumes cloud par pod
- Récupération du `Secret` créé automatiquement par le chart pour y référencer le mot de passe

### Étape 3 — Mongo Express

**Ressources déployées :** `Deployment`, `Service` (ClusterIP)

**Concepts démontrés :**
- Injection du mot de passe via `secretKeyRef` pour lire directement dans le `Secret` du chart
- Utilisation d'un `Service` headless pour cibler le pod primaire MongoDB via DNS (`mongodb-0.mongodb-headless`) plutôt qu'un VIP load-balancé
- Interpolation de variables d'environnement avec la syntaxe `$(VAR)` dans les specs de conteneur

### Étape 4 — Nginx Ingress Controller

**Ressources déployées :** `Deployment` (contrôleur Nginx), `Service` (LoadBalancer), `Ingress`

**Concepts démontrés :**
- Installation du contrôleur Nginx via Helm (OCI registry)
- Provisionnement automatique d'un **NodeBalancer Linode** lors de la création du Service `LoadBalancer`
- Règle `Ingress` avec `ingressClassName: nginx` pour router le trafic externe vers Mongo Express
- Test de persistance : scale down à 0 puis scale up à 3 — les volumes se réattachent automatiquement aux mêmes pods et les données sont conservées

## EN — Description

This project deploys a MongoDB + Mongo Express stack on a Linode managed Kubernetes cluster (LKE) using Helm to orchestrate the complex resources.

### Step 1 — LKE Cluster

Provisioning a managed Kubernetes cluster on Linode (2 nodes, 4 GB plan). Configuring `kubectl` via the `kubeconfig.yaml` file downloaded from the Linode console.

### Step 2 — MongoDB via Helm (Bitnami)

**Deployed resources:** `StatefulSet` (3 replicas + 1 arbiter), headless `Service`, `Secret`, `PersistentVolumeClaim` (one per pod)

**Concepts demonstrated:**
- Overriding default chart values via a custom `values.yaml` file
- `replicaset` mode: 3 data instances + 1 arbiter for primary elections
- `storageClass: linode-block-storage`: dynamic cloud volume provisioning per pod
- Using the `Secret` automatically created by the chart to reference the password

### Step 3 — Mongo Express

**Deployed resources:** `Deployment`, `Service` (ClusterIP)

**Concepts demonstrated:**
- Password injection via `secretKeyRef` to read directly from the chart's `Secret`
- Using a headless `Service` to target the primary MongoDB pod by DNS (`mongodb-0.mongodb-headless`) rather than a load-balanced VIP
- Environment variable interpolation with `$(VAR)` syntax in container specs

### Step 4 — Nginx Ingress Controller

**Deployed resources:** `Deployment` (Nginx controller), `Service` (LoadBalancer), `Ingress`

**Concepts demonstrated:**
- Installing the Nginx controller via Helm (OCI registry)
- Automatic provisioning of a **Linode NodeBalancer** when the `LoadBalancer` Service is created
- `Ingress` rule with `ingressClassName: nginx` to route external traffic to Mongo Express
- Persistence test: scale down to 0 then back to 3 — volumes reattach automatically to the same pods and data is preserved

---

## FR — Démarrage rapide

```bash
# 1. Pointer kubectl vers le cluster LKE
chmod 400 your-kubeconfig.yaml
export KUBECONFIG=your-kubeconfig.yaml

# 2. Déployer MongoDB
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install mongodb --values helm-mongodb.yaml bitnami/mongodb

# 3. Déployer Mongo Express
kubectl apply -f helm-mongo-express.yaml

# 4. Installer le contrôleur Nginx Ingress
helm install nginx-ingress oci://ghcr.io/nginx/charts/nginx-ingress \
  --version 2.5.1 \
  --set controller.reportIngressStatus.enable=true

# 5. Appliquer la règle Ingress (mettre à jour host dans helm-ingress.yaml)
kubectl apply -f helm-ingress.yaml
```

## EN — Quick Start

```bash
# 1. Point kubectl to the LKE cluster
chmod 400 your-kubeconfig.yaml
export KUBECONFIG=your-kubeconfig.yaml

# 2. Deploy MongoDB
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install mongodb --values helm-mongodb.yaml bitnami/mongodb

# 3. Deploy Mongo Express
kubectl apply -f helm-mongo-express.yaml

# 4. Install Nginx Ingress Controller
helm install nginx-ingress oci://ghcr.io/nginx/charts/nginx-ingress \
  --version 2.5.1 \
  --set controller.reportIngressStatus.enable=true

# 5. Apply Ingress rule (update host in helm-ingress.yaml first)
kubectl apply -f helm-ingress.yaml
```

---

## Project Structure

```
.
├── helm-mongodb.yaml          # Helm values: MongoDB ReplicaSet, block storage, auth
├── helm-mongo-express.yaml    # Deployment + ClusterIP Service for Mongo Express
└── helm-ingress.yaml          # Ingress rule routing external traffic to Mongo Express
```
