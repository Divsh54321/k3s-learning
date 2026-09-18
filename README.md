# Kubernetes Fundamentals with k3s

Project 4 (Stage A) of a hands-on DevOps learning track. This project focuses purely on core Kubernetes concepts — Pods, Deployments, Services, self-healing, rolling updates — using **k3s**, a lightweight Kubernetes distribution, deliberately kept separate from AWS-specific complexity (EKS comes later, as Stage B).

The goal: understand *Kubernetes itself* first, so that when AWS-specific abstractions (IAM roles for pods, VPC networking, load balancer controllers) get layered on top in Stage B, it's clear what's Kubernetes behavior versus what AWS is adding.

---

## What this deploys

- A single-node k3s cluster on one EC2 instance
- An `nginx` Deployment with 3 replicas
- A `NodePort` Service exposing nginx on port `30080`

Deliberately simple — nginx needs no custom code, so all the focus stays on Kubernetes mechanics rather than application logic.

---

## Setup

### 1. Provision a server

Any Ubuntu 22.04/24.04 EC2 instance works. `t3.small` (2GB RAM) is recommended over `t3.micro` — k3s runs a full control plane plus workloads on one machine, and 1GB is tight enough to cause resource-starvation symptoms (see the debugging notes below).

**Security group must allow, in addition to SSH (22):**
| Port range | Purpose |
|---|---|
| 6443 | Kubernetes API server |
| 30000-32767 | NodePort range (for exposing apps) |

### 2. Install k3s

```bash
curl -sfL https://get.k3s.io | sh -
```

This installs and starts an entire single-node Kubernetes control plane (API server, embedded datastore, scheduler, controller manager) plus a `kubelet` and `containerd`, bundled into one process.

### 3. Set up kubectl

k3s's `kubectl` defaults to reading `/etc/rancher/k3s/k3s.yaml` (root-only), not the usual `~/.kube/config`. Fix both the permissions and the default lookup path:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
chmod 600 ~/.kube/config

export KUBECONFIG=~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
```

Verify:
```bash
kubectl get nodes
```

### 4. Deploy nginx

`nginx-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

`nginx-service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

```bash
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-service.yaml
```

Visit `http://<EC2_PUBLIC_IP>:30080` — you should see the nginx welcome page.

---

## Concepts demonstrated hands-on

- **Self-healing**: delete a Pod manually (`kubectl delete pod <name>`) and watch the Deployment create a replacement within seconds — no script, no manual intervention. This is Kubernetes' reconciliation loop: it continuously compares desired state (3 replicas) against actual state, and corrects any gap.
- **Live scaling**: `kubectl scale deployment nginx-deployment --replicas=5` changes replica count on the fly, no YAML edit needed.
- **Rolling updates**: changing the image tag in the Deployment and re-applying triggers a gradual rollout — new Pods come up and become healthy *before* old Pods are terminated, so the Service always has enough healthy backends to serve traffic. Zero downtime, and it's the same reconciliation mechanism as self-healing, just applied gradually.
- **Rollback**: `kubectl rollout undo deployment/nginx-deployment` reverts to the previous version instantly if a rollout goes wrong.

---

## Debugging notes, kept here on purpose

Two real issues came up, worth documenting:

1. **Resource starvation on startup.** Right after installing k3s, `kubectl` calls started timing out (`TLS handshake timeout`, `keepalive ping failed`) across multiple internal components simultaneously. `free -h` and `top` showed heavy `wa` (I/O wait) and near-total memory usage — the classic signature of a machine too resource-constrained to keep up, not a broken install. This settled down once the instance had adequate memory (2GB) and finished its initial startup load (image pulls, Traefik/servicelb add-ons starting). For tighter resource budgets, k3s can be installed without its extra add-ons:
   ```bash
   curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik --disable=servicelb" sh -
   ```

2. **Security group drift.** The app wasn't reachable on the NodePort despite the Deployment and Service both looking correct. Root cause: a Terraform `apply` had been run from the *wrong* folder (the original Project 1/2 infra folder), so instance-type changes landed on the *existing* server instead of provisioning a genuinely new, separate one with its own security group — meaning the new NodePort/API-server firewall rules were never actually created. Lesson: always confirm *which* Terraform state you're actually modifying (`terraform state list`, or just double-check your current directory) before assuming a change went where intended. Verified directly via:
   ```bash
   aws ec2 describe-instances --region <region> --query "Reservations[*].Instances[*].{Name:Tags[?Key=='Name']|[0].Value, SGs:SecurityGroups}" --output table
   ```

---

## Project Status

- [x] Single-node k3s cluster running
- [x] nginx Deployment + NodePort Service reachable externally
- [x] Self-healing demonstrated (manual Pod deletion → automatic replacement)
- [x] Live scaling demonstrated
- [x] Rolling update + rollback demonstrated
- [ ] Stage B: same concepts on AWS EKS (IAM roles for pods, VPC networking, load balancer controller)
- [ ] Multi-node cluster (Scheduler behavior with node choice)
