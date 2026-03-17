# Automated Kubernetes Cluster Deployment Using Ansible

> **Complete DevOps Lab Guide** — Ubuntu + Docker + Ansible + Kubernetes

Deploy a fully automated, multi-node Kubernetes cluster on a single Ubuntu laptop using Docker containers to simulate real server nodes and Ansible playbooks to handle all configuration.

---

## What You'll Build

- **1 Ansible control node** container that orchestrates everything via SSH
- **1 Kubernetes master** (control-plane) node container
- **2 Kubernetes worker** node containers
- All nodes connected over a custom Docker bridge network (`k8s-net`)
- A working Kubernetes cluster initialised by `kubeadm`
- A sample **Nginx** application deployed as a workload

---

## Architecture

```
╔══════════════════════════════════════════════════════════════╗
║                  DOCKER HOST (Ubuntu Laptop)                 ║
║  ┌─────────────────────────────────────────────────────┐    ║
║  │         k8s-net  (172.20.0.0/16 bridge)             │    ║
║  │                                                      │    ║
║  │  ┌───────────────┐  SSH + Ansible Playbooks          │    ║
║  │  │ ansible-ctrl  │──────────────────────────┐        │    ║
║  │  │  172.20.0.10  │                          ▼        │    ║
║  │  └───────────────┘             ┌──────────────────┐  │    ║
║  │                                │   k8s-master     │  │    ║
║  │                                │   172.20.0.11    │  │    ║
║  │                                │ (control-plane)  │  │    ║
║  │                                └────────┬─────────┘  │    ║
║  │                                         │ kubeadm    │    ║
║  │                            ┌────────────┴──────────┐ │    ║
║  │                    ┌───────┴────────┐  ┌───────────┴──┐  ║
║  │                    │  k8s-worker1   │  │ k8s-worker2  │  ║
║  │                    │  172.20.0.12   │  │ 172.20.0.13  │  ║
║  │                    └────────────────┘  └──────────────┘  ║
║  └─────────────────────────────────────────────────────────┘ ║
╚══════════════════════════════════════════════════════════════╝
```

| Container | IP | Role |
|---|---|---|
| `ansible-ctrl` | 172.20.0.10 | Ansible automation control node |
| `k8s-master` | 172.20.0.11 | Kubernetes control-plane |
| `k8s-worker1` | 172.20.0.12 | Worker node 1 |
| `k8s-worker2` | 172.20.0.13 | Worker node 2 |

---

## Prerequisites

| Requirement | Details |
|---|---|
| OS | Ubuntu 20.04 / 22.04 / 24.04 LTS (64-bit) |
| RAM | Minimum 8 GB (16 GB recommended) |
| CPU | 4+ cores recommended |
| Disk | At least 20 GB free |
| Internet | Required for package downloads |
| Privileges | `sudo` / root access on the host |

---

## Project Structure

```
k8s-ansible-lab/
├── Dockerfile                        # Node image with SSH
├── docker-compose.yml                # Optional compose file
├── setup.sh                          # One-shot setup script
├── ansible/
│   ├── inventory/
│   │   └── hosts.ini                 # Inventory file
│   ├── group_vars/
│   │   └── all.yml                   # Shared variables
│   ├── roles/
│   │   ├── common/                   # Base packages
│   │   ├── container-runtime/
│   │   ├── kubernetes/
│   │   ├── k8s-master/
│   │   └── k8s-worker/
│   ├── site.yml                      # Master playbook
│   ├── 01-prerequisites.yml
│   ├── 02-container-runtime.yml
│   ├── 03-kubernetes.yml
│   ├── 04-master-init.yml
│   └── 05-worker-join.yml
└── apps/
    └── nginx-deployment.yml          # Sample app
```

---

## Quick Start

### 1. Clone / create the project directory

```bash
mkdir -p k8s-ansible-lab && cd k8s-ansible-lab
```

### 2. Install Docker on the host

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo $VERSION_CODENAME) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER && newgrp docker
docker --version
```

### 3. Install Ansible on the host (optional)

```bash
sudo apt-get install -y python3-pip
pip3 install ansible
ansible --version
```

### 4. Run the one-shot setup script

```bash
chmod +x setup.sh
./setup.sh
```

This script builds the Docker image, creates the `k8s-net` network, and starts all four containers.

### 5. Enter the Ansible control node

```bash
docker exec -it ansible-ctrl bash
```

### 6. Set up SSH key distribution

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
ssh-copy-id ansible@172.20.0.11   # k8s-master
ssh-copy-id ansible@172.20.0.12   # k8s-worker1
ssh-copy-id ansible@172.20.0.13   # k8s-worker2
```

### 7. Run the Ansible playbooks

```bash
cd /ansible

# Test connectivity
ansible all -i inventory/hosts.ini -m ping

# Full cluster deployment
ansible-playbook -i inventory/hosts.ini site.yml

# Or step-by-step
ansible-playbook -i inventory/hosts.ini 01-prerequisites.yml
ansible-playbook -i inventory/hosts.ini 02-container-runtime.yml
ansible-playbook -i inventory/hosts.ini 03-kubernetes.yml
ansible-playbook -i inventory/hosts.ini 04-master-init.yml
ansible-playbook -i inventory/hosts.ini 05-worker-join.yml
```

### 8. Verify the cluster

```bash
kubectl get nodes -o wide
kubectl get pods --all-namespaces
```

### 9. Deploy the sample Nginx app

```bash
kubectl apply -f /apps/nginx-deployment.yml
kubectl get pods
kubectl get svc
```

---

## Useful kubectl Commands

| Action | Command |
|---|---|
| List all nodes | `kubectl get nodes -o wide` |
| List all pods | `kubectl get pods --all-namespaces` |
| Describe a pod | `kubectl describe pod <name>` |
| View pod logs | `kubectl logs <pod-name>` |
| Execute into pod | `kubectl exec -it <pod-name> -- bash` |
| Apply manifest | `kubectl apply -f file.yml` |
| Delete resource | `kubectl delete -f file.yml` |
| Get services | `kubectl get svc` |
| Cluster info | `kubectl cluster-info` |
| Watch events | `kubectl get events --sort-by=.metadata.creationTimestamp` |

---

## Troubleshooting

**kubeadm init fails**
```bash
kubeadm reset
rm -rf /var/lib/etcd
kubeadm init --apiserver-advertise-address=172.20.0.11 \
  --pod-network-cidr=192.168.0.0/16 \
  --ignore-preflight-errors=all --v=5
```

**Pods stuck in Pending**
```bash
kubectl describe pod <pod-name>
kubectl describe nodes | grep -A5 Allocatable
kubectl get pods -n kube-system | grep -E 'calico|flannel|weave'
```

**SSH connection refused**
```bash
docker exec k8s-master service ssh status
docker exec k8s-master service ssh restart
docker inspect k8s-master | grep IPAddress
```

**Ansible connectivity failures**
```bash
ansible all -i inventory/hosts.ini -m ping -vvv
ssh -i ~/.ssh/id_rsa ansible@172.20.0.11 -v
```

> **Tip:** Always run `ansible-playbook` with `-v` or `-vvv` when troubleshooting to see exactly what Ansible is executing on each node.

---

## Tech Stack

- **Ubuntu** — Host OS
- **Docker** — Container runtime for simulating multi-node environment
- **Ansible** — Configuration management and automation
- **Kubernetes** — Container orchestration (`kubeadm` init)
- **Calico / CNI** — Pod networking

---

*Automated Kubernetes Cluster Deployment Using Ansible — Complete DevOps Lab Guide*