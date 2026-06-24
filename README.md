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

`StatefulSet` (3 réplicas + 1 arbiter), `Service` headless, `Secret`, `PersistentVolumeClaim` (un par pod).

Les valeurs par défaut du chart Bitnami sont surchargées via un fichier `values.yaml` personnalisé. Le mode `replicaset` déploie 3 instances de données + 1 arbiter pour les élections de primaire. La `storageClass: linode-block-storage` provisionne dynamiquement un volume cloud par pod. Le mot de passe est lu depuis le `Secret` créé automatiquement par le chart.

### Étape 3 — Mongo Express

`Deployment` + `Service` ClusterIP.

Le mot de passe est injecté via `secretKeyRef` pour lire directement dans le `Secret` du chart Bitnami. La connexion à MongoDB utilise un `Service` headless pour cibler le pod primaire par son DNS stable (`mongodb-0.mongodb-headless`) plutôt qu'un VIP load-balancé. Les variables d'environnement entre elles utilisent la syntaxe `$(VAR)` dans les specs de conteneur.

### Étape 4 — Nginx Ingress Controller

`Deployment` contrôleur Nginx, `Service` LoadBalancer, `Ingress`.

Le contrôleur Nginx est installé via Helm (OCI registry). La création du `Service` de type `LoadBalancer` provisionne automatiquement un **NodeBalancer Linode**. La règle `Ingress` avec `ingressClassName: nginx` route le trafic externe vers Mongo Express. Test de persistance : scale down à 0 puis scale up à 3 — les volumes se réattachent aux mêmes pods et les données sont conservées.

---

## EN — Description

This project deploys a MongoDB + Mongo Express stack on a Linode managed Kubernetes cluster (LKE) using Helm to orchestrate the complex resources.

### Step 1 — LKE Cluster

Provisioning a managed Kubernetes cluster on Linode (2 nodes, 4 GB plan). `kubectl` is configured via the `kubeconfig.yaml` file downloaded from the Linode console.

### Step 2 — MongoDB via Helm (Bitnami)

`StatefulSet` (3 replicas + 1 arbiter), headless `Service`, `Secret`, `PersistentVolumeClaim` (one per pod).

Default chart values are overridden via a custom `values.yaml`. The `replicaset` mode runs 3 data instances + 1 arbiter for primary elections. `storageClass: linode-block-storage` dynamically provisions a cloud volume per pod. The password is read from the `Secret` that the chart creates automatically.

### Step 3 — Mongo Express

`Deployment` + `ClusterIP` `Service`.

Password injection uses `secretKeyRef` to read directly from the Bitnami chart's `Secret`. The MongoDB connection targets the primary pod via its stable DNS name (`mongodb-0.mongodb-headless`) using a headless `Service`, rather than a load-balanced VIP. Environment variable interpolation between vars uses the `$(VAR)` syntax in container specs.

### Step 4 — Nginx Ingress Controller

Nginx controller `Deployment`, `LoadBalancer` `Service`, `Ingress` rule.

The Nginx controller is installed via Helm from the OCI registry. Creating the `LoadBalancer` Service automatically provisions a **Linode NodeBalancer**. An `Ingress` rule with `ingressClassName: nginx` routes external traffic to Mongo Express. Persistence test: scale down to 0 then back to 3 — volumes reattach to the same pods and data is preserved.

---

## Project Structure

```
.
├── helm-mongodb.yaml          # Helm values: MongoDB ReplicaSet, block storage, auth
├── helm-mongo-express.yaml    # Deployment + ClusterIP Service for Mongo Express
└── helm-ingress.yaml          # Ingress rule routing external traffic to Mongo Express
```
