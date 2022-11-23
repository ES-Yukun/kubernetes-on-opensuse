# kubernetes-on-opensuse

__**OS**__
OpenSUSE Leap 15.4

__**First**__
You need to disable **SWAP**
```bash
sudo swapoff -a
```
And after edit `/etc/fstab`

__**Second (master)**__
Run As **root**
`install.sh`
```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sudo sysctl --system
zypper addrepo --type yum --gpgcheck-strict --refresh https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64 google-k8s
rpm --import https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
rpm --import https://packages.cloud.google.com/yum/doc/yum-key.gpg
zypper refresh google-k8s
echo -e "a\n" | zypper ref
zypper -n in containerd cri-tools conntrack-tools
sudo mkdir -p /etc/containerd
systemctl disable --now firewalld
systemctl enable --now containerd
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
vm.overcommit_memory = 1
vm.panic_on_oom = 0
kernel.panic = 10
kernel.panic_on_oops = 1
kernel.keys.root_maxkeys = 1000000
kernel.keys.root_maxbytes = 25000000
EOF
sysctl --system
sudo mkdir -p /etc/containerd
sudo containerd config default > /etc/containerd/config.toml
if grep -q "SystemdCgroup = true" "/etc/containerd/config.toml"; then
  echo "Config found, skip rewriting..."
else
  sed -i -e "s/SystemdCgroup \= false/SystemdCgroup \= true/g" /etc/containerd/config.toml
fi
zypper in kubelet kubeadm kubectl
systemctl enable --now kubelet
cat > ~/init_kubelet.yaml <<EOF
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
bootstrapTokens:
- token: "$(openssl rand -hex 3).$(openssl rand -hex 8)"
  description: "kubeadm bootstrap token"
  ttl: "24h"
nodeRegistration:
  criSocket: "/var/run/containerd/containerd.sock"
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
controllerManager:
  extraArgs:
    bind-address: "0.0.0.0" # Used by Prometheus Operator
scheduler:
  extraArgs:
    bind-address: "0.0.0.0" # Used by Prometheus Operator
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: "systemd"
protectKernelDefaults: true
EOF
kubeadm init --config init_kubelet.yaml
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium \
    --namespace kube-system
```

__**Second (worker)**__
Run As **root**
`install.sh`
```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sudo sysctl --system
zypper addrepo --type yum --gpgcheck-strict --refresh https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64 google-k8s
rpm --import https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
rpm --import https://packages.cloud.google.com/yum/doc/yum-key.gpg
zypper refresh google-k8s
echo -e "a\n" | zypper ref
zypper -n in containerd cri-tools conntrack-tools
sudo mkdir -p /etc/containerd 
systemctl disable --now firewalld
systemctl enable --now containerd
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
vm.overcommit_memory = 1
vm.panic_on_oom = 0
kernel.panic = 10
kernel.panic_on_oops = 1
kernel.keys.root_maxkeys = 1000000
kernel.keys.root_maxbytes = 25000000
EOF
sysctl --system
sudo mkdir -p /etc/containerd
sudo containerd config default > /etc/containerd/config.toml
if grep -q "SystemdCgroup = true" "/etc/containerd/config.toml"; then
  echo "Config found, skip rewriting..."
else
  sed -i -e "s/SystemdCgroup \= false/SystemdCgroup \= true/g" /etc/containerd/config.toml
fi
zypper in kubelet kubeadm kubectl
systemctl enable --now kubelet
```
**Ignoring conntrack breakout, just pick**
```Solution 2: break kubelet-1.15.4-0.x86_64 by ignoring some of its dependencies Choose from above solutions by number or skip, retry or cancel [1/2/s/r/c] (c): 2 ...

Solution 3: break kubelet-1.13.3-0.x86_64 by ignoring some of its dependencies Choose from above solutions by number or skip, retry or cancel [1/2/3/s/r/c] (c): 3```

