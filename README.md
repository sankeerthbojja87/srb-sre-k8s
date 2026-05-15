# SRB SRE Kubernetes Practice Lab

Hands-on Kubernetes, Helm, and SRE operations practice for a local Mac environment.

This repo captures a practical interview-prep lab built with:

- Colima for a local Linux container runtime on macOS
- Docker as the container runtime
- kind for a local multi-node Kubernetes cluster
- kubectl for cluster operations
- NGINX as a sample application workload
- NGINX Ingress Controller for local ingress practice
- Helm for chart/package management practice

## Local Cluster Shape

The cluster is created with `kind-multinode.yaml`:

```text
1 control-plane node
2 worker nodes
```

This lets you practice production-style workflows such as node drains, pod rescheduling, Pod Disruption Budgets, rolling updates, rollback, service routing, and ingress routing.

## Build The Cluster

Start Colima:

```bash
colima start --cpu 4 --memory 6 --disk 30
docker version
```

Create the cluster:

```bash
kind create cluster --name srb-sre-lab --config kind-multinode.yaml
kubectl get nodes
```

## Core Kubernetes App

Create the namespace:

```bash
kubectl create namespace srb-sre-lab
```

Apply the production-style deployment:

```bash
kubectl apply -f web-prod.yaml
```

Create the service if it does not already exist:

```bash
kubectl expose deployment web --port=80 --target-port=80 -n srb-sre-lab
```

Apply the Pod Disruption Budget:

```bash
kubectl apply -f web-pdb.yaml
```

Inspect:

```bash
kubectl get deploy,rs,pods,svc,pdb -n srb-sre-lab -o wide
kubectl describe deployment web -n srb-sre-lab
```

## Service Test

Test service routing from inside the cluster:

```bash
kubectl run test-client --rm -it --image=curlimages/curl -n srb-sre-lab -- sh
curl web
exit
```

Test from the Mac through port-forward:

```bash
kubectl port-forward svc/web 8080:80 -n srb-sre-lab
curl http://localhost:8080
```

## Rolling Update And Rollback

Roll forward:

```bash
kubectl set image deployment/web nginx=nginx:1.26 -n srb-sre-lab
kubectl rollout status deployment/web -n srb-sre-lab
kubectl get rs -n srb-sre-lab
```

Roll back:

```bash
kubectl rollout undo deployment/web -n srb-sre-lab
kubectl rollout status deployment/web -n srb-sre-lab
```

## Node Maintenance

Cordon a node:

```bash
kubectl cordon srb-sre-lab-worker2
```

Drain it while respecting the PDB:

```bash
kubectl drain srb-sre-lab-worker2 --ignore-daemonsets --delete-emptydir-data
```

Bring it back:

```bash
kubectl uncordon srb-sre-lab-worker2
```

Key learning: `cordon` prevents new pods from scheduling on the node. `drain` evicts existing application pods while respecting the Pod Disruption Budget.

## HPA

Create a Horizontal Pod Autoscaler:

```bash
kubectl apply -f web-hpa.yaml
kubectl get hpa -n srb-sre-lab
```

In kind, HPA CPU metrics require metrics-server. Without metrics-server, HPA will show `cpu: <unknown>/60%`.

## Ingress

The `web-ingress.yaml` routes `web.local` to the `web` Service.

Install an ingress controller, then apply:

```bash
kubectl apply -f web-ingress.yaml
```

Local test through ingress controller port-forward:

```bash
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8081:80
curl -H "Host: web.local" http://localhost:8081
```

## Helm

The `web-chart/` directory was created with:

```bash
helm create web-chart
```

Render and lint:

```bash
helm lint web-chart
helm template web-helm web-chart
```

Install:

```bash
helm install web-helm web-chart -n srb-sre-lab
```

Upgrade:

```bash
helm upgrade web-helm web-chart -n srb-sre-lab --set replicaCount=2
```

Rollback:

```bash
helm rollback web-helm 1 -n srb-sre-lab
```

## Interview Talking Point

This lab demonstrates:

- multi-node Kubernetes cluster setup
- Deployment, ReplicaSet, Pod relationships
- ClusterIP Service routing
- CoreDNS and kube-proxy behavior
- readiness and liveness probes
- resource requests and limits
- rolling update and rollback
- Pod Disruption Budget behavior during node drain
- topology spread constraints
- HPA object creation and metrics-server dependency
- Ingress routing through NGINX Ingress Controller
- Helm chart rendering, linting, install, upgrade, and rollback
