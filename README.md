# Building a Raspberry Pi Kubernetes Cluster on Ubuntu using kubeadm and Containerd

## Changing current ubuntu password

```bash
ssh ubuntu@<ip>
```

## Adding new user

```bash
ssh ubuntu@<ip>
sudo adduser k8s-user
```

## Adding Privilege Escalation

```bash
sudo visudo
```

```bash
k8s-user    ALL=(ALL:ALL) ALL
```

```bash
logout
```

## Creating a key pair on client server

```bash
ssh-keygen -t rsa
```

## Copying public key to the remote server

```bash
ssh-copy-id k8s-user@<ip>
```

## Updating users

```bash
ssh k8s-user@<ip>
sudo usermod -s /bin/nologin ubuntu
```

## Updating sshd config

```bash
sudo vi /etc/ssh/sshd_config
```

```bash
PermitRootLogin no
PasswordAuthentication no
```

```bash
sudo systemctl reload sshd
```

## Updating OS

```bash
sudo apt update && sudo apt dist-upgrade -y
```

## Configuring network

```bash
sudo vi /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

```bash
network: {config: disabled}
```

```bash
sudo vi /etc/netplan/01-netcfg.yaml
```

```bash
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses:
       - <ip>/24
      gateway4: 192.168.0.1
      nameservers:
        addresses: [192.168.0.1, 8.8.8.8]
        search: [domain.local]
```

```bash
sudo vi /etc/netplan/50-cloud-init.yaml
```

> Delete all text apart from the comments

```bash
sudo netplan apply
```

```bash
sudo hostnamectl set-hostname pimaster
```

> sudo hostnamectl set-hostname piworker - for worker node

```bash
sudo vi /etc/hosts
```

```bash
<control plane ip> pimaster
<worker ip> piworker
```

## Checking iptables version

```bash
iptables --version
```

> Should be iptables v1.8.4 (legacy)

OR

```bash
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
sudo update-alternatives --set arptables /usr/sbin/arptables-legacy
sudo update-alternatives --set ebtables /usr/sbin/ebtables-legacy
```

## Updating boot command

```bash
cgroup="$(head -n1 /boot/firmware/cmdline.txt) cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1 swapaccount=1"
echo $cgroup | sudo tee /boot/firmware/cmdline.txt

sudo reboot

cat /proc/cmdline
```

## Disabling swap

```bash
ssh k8s-user@<ip>

sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

## containerd prerequisites

```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Setup required sysctl params, these persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

## Installing containerd

```bash
sudo apt install containerd -y

sudo mkdir -p /etc/containerd

sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerds

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"

sudo apt install kubeadm kubelet kubectl -y
```

## Bootstrapping master

```bash
sudo kubeadm config images pull
sudo kubeadm init

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Installing CNI plugin

```bash
curl https://docs.projectcalico.org/manifests/calico-typha.yaml -o calico.yam
kubectl apply -f calico.yaml
```

## Bootstrapping worker

```bash
sudo kubeadm join <control plane ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

## Checking

```bash
kubectl get nodes -o wide
```

## Usefulness

### Join the additional node after the init token expired

```bash
kubeadm token create --print-join-command
```

### Installing containerd CLI

```bash
wget https://github.com/containerd/containerd/releases/download/v1.5.4/containerd-1.5.4-linux-amd64.tar.gz
tar xvf containerd-1.5.4-linux-amd64.tar.gz
cd bin

sudo ctr --namespace k8s.io container list
```
