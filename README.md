# Piep-Kubernetes
## Installation
### Kubernetes Setup
#### containerd:
```
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
> overlay
> br_netfilter
> EOF
```
```
sudo modprobe overlay
sudo modprobe br_netfilter
```

*# Setup required sysctl params, these persist across reboots.*
```
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
> net.bridge.bridge-nf-call-iptables  = 1
> net.ipv4.ip_forward                 = 1
> net.bridge.bridge-nf-call-ip6tables = 1
> EOF
```


*# Apply sysctl params without reboot*
```
sudo sysctl --system
```
#### Install containerd
```
sudo apt install containerd  
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
```
#### Use *systemd* for cgroup driver in /etc/containerd/config.toml
```
sudo nano /etc/containerd/config.toml

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```
```
sudo systemctl restart containerd
```

### Installing kubeadm, kubelet and kubectl
#### Debian-based distributions
* # Update the apt package index and install packages needed to use the Kubernetes apt repository: *
```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```



### MongoDB
