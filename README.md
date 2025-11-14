# README.md — On-Prem Kubernetes UAT: one-sb Demo Stack

> Purpose: concise, step-by-step summary of what we did (from infra → AWS ECR → cluster → deploy services & ingress), ready to paste into your repo.
> This file intentionally **excludes** troubleshooting steps — it lists the standard, successful actions and commands used to set the environment up.

---

## Overview

This repository contains manifests and instructions to deploy the **one-sb** UAT stack on an on-prem Kubernetes cluster.
Components deployed:

* Namespace: `insurance-demo`
* ConfigMap: `env-uat-cm` (application config / env)
* ServiceAccount: `onsurityk8s` (imagePullSecret attached)
* ECR image pull secret: `ecr-registry-secret` (refreshed by CronJob)
* External DB service: `one-sb-db` + Endpoints → external Postgres VM `192.168.1.141:5432`
  * `one-sb-demo` (backend)
  * `one-sb-health-demo` (frontend)
  * `one-sb-motor-demo` (frontend)
  * `one-sb-term-demo` (frontend)
* Services:

  * ClusterIP for DB (no selector) + Endpoints
  * NodePort/ClusterIP services for apps
* Ingress resources for domain routing
* `ingress-nginx` controller (bare-metal / NodePort)
* External NGINX Load Balancer VM that proxies to ingress controller NodePorts
* Optional CronJob to auto-refresh ECR secret every 6 hours

---

## Architecture (summary)

```
User Browser
   ↓ (DNS: uat-*.insurance.onsurity.com -> LB VM IP)
External nginx LB (VM)
   ↓ (proxy to worker-node:NodePort e.g. 30243)
Kubernetes nodes (worker-1, worker-2)
   ↓ (NodePort -> ingress-nginx controller)
ingress-nginx controller (namespace: ingress-nginx)
   ↓ (Ingress rules)
Service -> Pod
   ↓
App containers (image from AWS ECR)
Database: external Postgres VM at 192.168.1.141:5432 (accessed via Service+Endpoints)
```

---

## Files / Manifests (where to find them)

Place the files in `manifests/` (or your chosen folder):

* `namespace.yaml`
* `serviceaccount-onsurityk8s.yaml`
* `ecr-registry-secret` (created via `kubectl create secret docker-registry ...`)
* `env-uat-cm.yaml` (ConfigMap with all env variables)
* `one-sb-db-svc.yaml` (ClusterIP)
* `one-sb-db-endpoints.yaml` (Endpoints -> 192.168.1.141:5432)
* `deployment-one-sb-demo.yaml` (backend)
* `service-one-sb-demo-nodeport.yaml` (NodePort for backend)
* `deployment-one-sb-health.yaml`, `service-one-sb-health.yaml` (health frontend)
* `deployment-one-sb-motor.yaml`, `service-one-sb-motor.yaml` (motor frontend)
* `deployment-one-sb-term.yaml`, `service-one-sb-term.yaml` (term frontend)
* `ingress-one-sb-demo.yaml` (Ingress definitions for `uat-api` / `uat-term` / `uat-health` / `uat-motor`)
* `ingress-nginx` install: applied official bare-metal manifest
* `ecr-refresh-configmap.yaml` + `ecr-refresh-cronjob.yaml` (CronJob to refresh ECR secret)
* `rbac-ecr-secret.yaml` (Role + RoleBinding for CronJob to manage secrets)
* `nginx-lb.conf` (External NGINX LB config for all domains) — put on LB VM under `/etc/nginx/conf.d/`

---

## Prerequisites

* Kubernetes cluster (control-plane + workers) up and `kubectl` configured
* Worker nodes labeled:

  * `worker-1` → `node-role=worker-1`
  * `worker-2` → `node-role=worker-2`
* AWS CLI configured (for ECR auth) on the machine that will run the secret refresh (or the CronJob image)
* External NGINX VM reachable from clients and worker nodes
* Postgres DB reachable at `192.168.1.141:5432` (DB admin configured listen/pg_hba/firewall accordingly)

---

## Core commands we ran (in order)

> Run these on the machine with `kubectl` configured. Replace values (IPs, account ID, region) as needed.

### 1. Namespace

```bash
kubectl create namespace insurance-demo
```

### 2. ServiceAccount

```bash
kubectl apply -f serviceaccount-onsurityk8s.yaml
# (or)
kubectl create sa onsurityk8s -n insurance-demo
kubectl patch sa onsurityk8s -n insurance-demo \
  -p '{"imagePullSecrets":[{"name":"ecr-registry-secret"}]}'
```

### 3. Create ECR image pull secret (manual one-time)

```bash
AWS_REGION=ap-south-1
ECR_REG=106102357433.dkr.ecr.${AWS_REGION}.amazonaws.com

kubectl delete secret ecr-registry-secret -n insurance-demo --ignore-not-found
kubectl create secret docker-registry ecr-registry-secret \
  --namespace insurance-demo \
  --docker-server=${ECR_REG} \
  --docker-username=AWS \
  --docker-password="$(aws ecr get-login-password --region ${AWS_REGION})"
```

### 4. ConfigMap (app environment)

```bash
kubectl apply -f env-uat-cm.yaml -n insurance-demo
```

### 5. DB Service + Endpoints (external DB)

```bash
kubectl apply -f one-sb-db-svc.yaml -n insurance-demo
kubectl apply -f one-sb-db-endpoints.yaml -n insurance-demo
```

### 6. Node labeling

```bash
kubectl label node <worker-node-name> node-role=worker-1 --overwrite
kubectl label node <worker2-node-name> node-role=worker-2 --overwrite
```

### 7. Deployments & services

(Example for backend)

```bash
kubectl apply -f deployment-one-sb-demo.yaml -n insurance-demo
kubectl apply -f service-one-sb-demo-nodeport.yaml -n insurance-demo
```

Repeat for other apps (health, motor, term).

### 8. Install ingress-nginx (bare-metal / NodePort)

```bash
kubectl create namespace ingress-nginx
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml
# verify NodePorts (example output had 80:30243,443:31481)
kubectl get svc -n ingress-nginx -o wide
```

### 9. Ingress resources (in `insurance-demo` namespace)

```bash
kubectl apply -f ingress-one-sb-demo.yaml -n insurance-demo
```

### 10. External NGINX LB VM config

Create `/etc/nginx/conf.d/insurance-uat.conf` with upstreams pointing to worker node IPs and ingress NodePort (HTTP e.g. 30243):

```nginx
upstream ingress_backend {
    server <worker-1-ip>:30243;
    server <worker-2-ip>:30243;
}
server {
    listen 80;
    server_name uat-api.insurance.onsurity.com uat-term.insurance.onsurity.com uat-health.insurance.onsurity.com uat-motor.insurance.onsurity.com;
    location / {
        proxy_pass http://ingress_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Reload nginx:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

### 11. Create CronJob to refresh ECR pull secret (optional but recommended)

* Apply `ecr-refresh-configmap.yaml`
* Apply `ecr-refresh-cronjob.yaml`
* Apply RBAC `ecr-secret-rbac.yaml`

Verify CronJob:

```bash
kubectl get cronjobs -n insurance-demo
kubectl get jobs -n insurance-demo
kubectl logs job/<job-name> -n insurance-demo
```

---

## How to access services

### Direct NodePort (internal access)

* Backend: `http://<worker-1-ip>:30127`  (example NodePort)
* Motor: `http://<worker-2-ip>:32536`
* Term: `http://<worker-2-ip>:31206`
* Health: `http://<worker-2-ip>:30784`

### Browser (Production / UAT) — via DNS + external LB

Make DNS entries point to the external LB VM IP:

```
uat-api.insurance.onsurity.com  → <LB_VM_IP>
uat-motor.insurance.onsurity.com → <LB_VM_IP>
uat-term.insurance.onsurity.com  → <LB_VM_IP>
uat-health.insurance.onsurity.com → <LB_VM_IP>
```

Then browse:

```
http://uat-motor.insurance.onsurity.com
http://uat-api.insurance.onsurity.com
```

---

## Useful verification commands

```bash
kubectl get ns
kubectl get pods -n kube-system -o wide
kubectl get pods -n ingress-nginx -o wide
kubectl get pods -n insurance-demo -o wide
kubectl get svc -n ingress-nginx -o wide
kubectl get svc -n insurance-demo -o wide
kubectl describe pod <pod-name> -n insurance-demo
kubectl logs -n insurance-demo -l app=one-sb-demo --tail=200
```

---

## Security & best practices (short list)

* Move sensitive DB credentials to a `Secret` instead of whole ConfigMap (already supported).
* Use the CronJob to rotate ECR tokens (on-prem) or use IAM roles/credential provider for AWS nodes.
* Add TLS termination at LB (Let's Encrypt or custom cert).
* Use NetworkPolicies if you want to restrict pod egress to DB VM.
* Use monitoring (Prometheus/Grafana) + alerts for pod crashes and image pull failures.

---

## Example: quick “one-liners” to redeploy the whole stack

```bash
# in manifests directory
kubectl apply -f namespace.yaml
kubectl apply -f env-uat-cm.yaml
kubectl apply -f serviceaccount-onsurityk8s.yaml
kubectl apply -f one-sb-db-svc.yaml
kubectl apply -f one-sb-db-endpoints.yaml
kubectl apply -f deployment-one-sb-demo.yaml
kubectl apply -f service-one-sb-demo-nodeport.yaml
kubectl apply -f deployment-one-sb-health.yaml
kubectl apply -f service-one-sb-health.yaml
kubectl apply -f deployment-one-sb-motor.yaml
kubectl apply -f service-one-sb-motor.yaml
kubectl apply -f deployment-one-sb-term.yaml
kubectl apply -f service-one-sb-term.yaml
kubectl apply -f ingress-one-sb-demo.yaml
```

---

## Notes / Ownership

* Owner: `DevOps Team`
* Contact: `chandrasekhar@...` (or whichever contact you want to use)
* Last updated: `YYYY-MM-DD` (replace with current date)

---

## License / Repo

Add your company license or internal policy note here.

---

If you want, I can:

* produce this file as an actual `README.md` file and save it (give me the path),
* or generate the exact YAML manifest files from the snippets above.
