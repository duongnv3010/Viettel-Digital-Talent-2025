# Kubernetes Lab – Deploying a 2‑Node Cluster with **Kubespray**

---

## 0. Topology & Requirements

| Hostname    | Role(s)                                       | vCPU / RAM (min) | OS (tested)       | SSH Port |
| ----------- | --------------------------------------------- | ---------------- | ----------------- | -------- |
| **ansible** | Control node: runs Ansible, Docker, Kubespray | 2 vCPU / 4 GiB   | Ubuntu 24.04 LTS+ | _custom_ |
| **master1** | `kube‑control‑plane`, `etcd`                  | 2 vCPU / 4 GiB   | Ubuntu 24.04 LTS+ | _custom_ |
| **worker1** | `kube‑node`                                   | 2 vCPU / 4 GiB   | Ubuntu 24.04 LTS+ | _custom_ |

---

## 1. Prepare the Ansible Control Node

```bash
# On the ansible host
sudo apt update
sudo apt install -y git curl python3 python3-pip
pip3 install --user ansible-core==2.15  # Kubespray ≥2.28 needs Ansible 2.15+
# Docker is only needed to enter the official Kubespray container
sudo apt install -y docker.io
```

## 2. Setup SSH key on all node

1. Use the ssh-keygen command to generate a public and private authentication key pair

```
ssh-keygen -t rsa
```

output:

<p align="center">
  <img src="assets\step2-1.png" alt="step2-1.png" width="600"/>
</p>

2. Copy key to remaining nodes

```
ssh-copy-id user@server1
```

Replace user@server1 with real user and IP

Output:

<p align="center">
  <img src="assets\step2-2.png" alt="step2-2.png" width="600"/>
</p>

<p align="center">
  <img src="assets\step2-3.png" alt="step2-3.png" width="600"/>
</p>

3. Test SSH

<p align="center">
  <img src="assets\step2-4.png" alt="step2-4.png" width="600"/>
</p>

<p align="center">
  <img src="assets\step2-5.png" alt="step2-5.png" width="600"/>
</p>

## 3. Get Kubespray

```

cd ~
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray

```

Output:

<p align="center">
  <img src="assets\step3.png" alt="step3.png" width="600"/>
</p>

## 4. Launch the Kubespray utility container (recommended)

```

docker run --rm -it \
 --mount type=bind,source="$(pwd)"/inventory/sample,dst=/inventory \
  --mount type=bind,source="$HOME"/.ssh/id_rsa,dst=/root/.ssh/id_rsa \
 quay.io/kubespray/kubespray:v2.28.0 bash

# We are now inside the container shell at /kubespray

```

output:

<p align="center">
  <img src="assets\step4.png" alt="step4.png" width="600"/>
</p>

## 5. Edit inventory file

Edit content in /inventory/inventory.ini\
Example inventory.ini:

```

[kube_control_plane]
master1 ansible_host=<MASTER_IP> ansible_port=<MASTER_SSH_PORT> ansible_user=<ssh_user>

[etcd:children]
kube_control_plane

[kube_node]
worker1 ansible_host=<WORKER_IP> ansible_port=<WORKER_SSH_PORT> ansible_user=<ssh_user>

[k8s_cluster:children]
kube_control_plane
kube_node

```

Output:

<p align="center">
  <img src="assets\step5.png" alt="step5.png" width="600"/>
</p>

## 6. Run the Ansible Playbook

```
ansible-playbook -i /inventory/inventory.ini cluster.yml --become --ask-pass --ask-become-pass
```

Enter SSH password if any or enter if step 2 has been performed.

A full deployment on two small VMs typically takes 10 – 15 min.
No task should end in FAILED. If you see failures, re‑run after fixing the cause; Kubespray is idempotent.

Output:

<p align="center">
  <img src="assets\step6.png" alt="step6.png" width="600"/>
</p>

## 7. Install Kubectl on Ansible Node

```
curl -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

Output:

<p align="center">
  <img src="assets\step7.png" alt="step7.png" width="600"/>
</p>

## 8. Kubeconfig configuration

On Master1 Node

```
sudo cat /etc/kubernetes/admin.conf
```

Copy that content to the ansible machine, paste it into the file named k8s-config.yaml. Then edit the line:

```
server: https://127.0.0.1:6443
```

to

```
server: https://<ip-master1>:6443
```

Output:

<p align="center">
  <img src="assets\step8.png" alt="step8.png" width="600"/>
</p>

## 9. Verify Installation

On Ansible Node:

```
export KUBECONFIG=k8s-config.yaml
```

then

```
kubectl get nodes -o wide
```

output:

<p align="center">
  <img src="assets\step9-1.png" alt="step9-1.png" width="600"/>
</p>

and

```
kubectl get pods -A -o wide
```

output:

<p align="center">
  <img src="assets\step9-2.png" alt="step9-2.png" width="600"/>
</p>
