# My Kubeadm cheatsheet

## Setup nodes (CP and Worker)

This cheatsheet is working with Ubuntu 20.04

### If needed

Update apt repo:

```bash
sudo apt update
```

Install curl and stuff:

```bash
sudo apt install -y apt-transport-https ca-certificates curl
```

### Setup Container runtime

Add key to use Docker repo:

```bash
sudo mkdir /etc/apt/keyrings/ && curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

Add Docker repo:

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install containerd as container runtime:

```bash
sudo apt update && sudo apt install containerd.io -y
```

Remove default containerd default configuration:

```bash
sudo rm -rf /etc/containerd/config.toml
```

Start containerd service:

```bash
sudo systemctl restart containerd
```

### Setup Kubeadm

Add K8s repo key:

```bash
sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://dl.k8s.io/apt/doc/apt-key.gpg
```

Add K8s repo:
```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Install kubeamd and kubelet:

```bash
sudo apt update && sudo apt install -y kubelet kubeadm
```

Enable kubelet service with systemd:

```bash
sudo systemctl enable --now kubelet
```

**Needed in case of reboot**

### Customize OS

Set containerd needed kernel modules:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

**Needed in case of reboot**

Activate containerd needed kernel modules:

```bash
sudo modprobe overlay && sudo modprobe br_netfilter
```

Set network kernel capabilities:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

Activate network kernel capabilities:

```bash
sudo sysctl --system
```

### Deactivate Swap ###

Check if you have any swap activated:

```bash
free -m
```

If swap is not set to zero deactivate it:

```bash
sudo swapoff -a
```

Remove swap definition from /etc/fstab, this is needed in case of reboot.

## Deploy Kubernetes

### Control plane nodes

If no specific configuration needed (network range for pod, ...):

```bash
sudo kubeadm init
```

When the command ends copy the kubeadm command at the end.

Get the kubeconfig content from the file /etc/kubernetes/admin.conf

### Worker nodes

Execute the kubeadm command copied from the controller deployment:

```bash
sudo kubeadm join 172.27.0.150:6443 --token fdadzt.2qr99d5n2boe1q2i --discovery-token-ca-cert-hash sha256:994bc9a0d431b3fadeae932961f147071151ab058af3ae744408d50416da1e70
```

## Install Kubernetes network

I'll use [Cilium](https://docs.cilium.io/en/stable/installation/k8s-install-helm/) from now on as CNI.

**at this point, [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) and [helm](https://helm.sh/docs/intro/install/) should be installed on your adminstration workstation**

Add Cilium Helm repo:

```bash
helm repo add cilium https://helm.cilium.io/
```

Deploy latest Cilium version through Helm with default settings:

```bash
helm install cilium cilium/cilium --version 1.14.0 --namespace kube-system
```

Restart pods that needs it for with Cilium:

```bash
kubectl get pods --all-namespaces -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,HOSTNETWORK:.spec.hostNetwork --no-headers=true | grep '<none>' | awk '{print "-n "$1" "$2}' | xargs -L 1 -r kubectl delete pod
```

With Cilium command line check the status of the CNI:

```bash
cilium status --wait
```

Check the connectivity of Cilium within the cluster:

```bash
cilium connectivity test
```

### Storage

Depending on your installation type.
If you are using a cloud provider, check the cloud provider K8s storage provisioning.
If you are on prem without tool dedicated for dynamic provisioning check Rook deployment.