# Piep-Kubernetes
## Installation
### Kubernetes Setup
#### containerd:
```
sudo apt install containerd  
```
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
*# Update the apt package index and install packages needed to use the Kubernetes apt repository:*
```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

*# Download the Google Cloud public signing key:*
```
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```

*# Add the Kubernetes apt repository:*
```
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

*# Update apt package index, install kubelet, kubeadm and kubectl, and pin their version:*
```
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Setup Configuration File
#### Create configuration file *config.yaml*

```
sudo nano config.yaml
```
```
apiVersion : kubelet.config.k8s.io/v1beta1
kind : KubeletConfiguration
cgroupDriver : systemd
---
apiVersion : kubeadm.k8s.io/v1beta2
kind : ClusterConfiguration
networking :
  podSubnet : "192.168.0.0/16"
```

### Initialize Cluster
```
sudo kubeadm init --config ./config.yaml
```

### Add piep-redis-service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: piep-redis
spec:
  type: NodePort
  selector:
    app: piep-redis
  ports:
    - port: 3000
      targetPort: 3000
      nodePort: 30007
```

*# Apply .yaml*
```
kubectl apply -f piep-redis-service.yaml
```

### *Only for Worker*
```
kubeadm join 172.23.1.116:6443 --token ebc733.7ysxvxlfgw3qxpjl \ --discovery-token-ca-cert-hash sha256:70630a919eecc923790b2310a3448c07155a9efa6990a1873a96fdd4ced7c98c
```

### Install Calico with Kubernetes API: *50 nodes or less*
*# Download the Calico networking manifest for the Kubernetes API datastore:*
```
curl https://docs.projectcalico.org/manifests/calico.yaml -O
```
*# Apply the manifest using the following command:*
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
```
kubectl apply -f calico.yaml
```

### Kubernetes Metrics Server
*# Install Metrics Server*
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
*# Change deployment file*
```
kubectl -n kube-system edit deploy metrics-server
```
Press *i* inside Vi file to edit it. If done, press *ESC*. 
Don't close the file! Press *:" and type *wq*

*# Add following command under imagePullPolicy*
```
command:
- /metrics-server
- --kubelet-insecure-tls
```

### Horizonal Pod Autoscaler
*# Create following .yaml*
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: piep-redis-deployment
spec:
  selector:
    matchLabels:
      app: piep-redis
  replicas: 1
  template:
    metadata:
      labels:
        app: piep-redis
    spec:
      containers:
      - name: piep-redis
        image: ralfluebben/piep-redis:1.27
        ports:
        - containerPort: 3000
        resources:
          limits:
            cpu: 1000m
          requests:
            cpu: 200m
        env:
        - name: db_addr
          value: "g2-mongo.wi.fh-flensburg.de"
```
```
kubectl apply -f piep-redis-deployment.yaml
```
*# Create HPA*
```
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```

*# Check current status of autoscaler:*
```
kubectl get hpa
```

*# Delete HPA*
```
kubectl delete hpa piep-redis-deployment
```


### Docker Installation
*Clone Piep*
```
git clone https://gitlab.com/ralfluebben/piep/
```
*# Step 1 - Installing Docker*
```
sudo apt update
```
```
sudo apt install apt-transport-https ca-certificates curl software-properties-common
```
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```
```
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
```
```
sudo apt update
```
```
apt-cache policy docker-ce
```
#### Finally install Docker
```
sudo apt install docker-ce
```
```
sudo systemctl status docker
```
#### Docker Images
*# Execute the following command to download the official ubuntu image to your computer:*
```
sudo docker pull ubuntu
```
*# View all images:*
```
sudo docker images
```
*# Create dockerfile*
```
sudo nano dockerfile
```
```
FROM ubuntu:20.04

RUN apt-get update
RUN apt-get -y install curl
RUN apt-get -y install nodejs
RUN apt-get -y install npm

RUN npm install --save express \
    && npm install --save qr-image \
    && npm install --save crypto \
    && npm install --save avatar-builder \
    && npm install --save mongodb

RUN apt-get install -y git

RUN git clone https://gitlab.com/ralfluebben/piep

WORKDIR /piep

CMD [ "node", "piep.js"]
```
*# Build image from dockerfile*
```
sudo docker build -t dockerfile . 
```
#### Push on DockerHub
*# Rename image*
```
sudo docker image tag 406faaf8f8b4 papazeus42/piep:latest
```
*# log into DockerHub from terminal*
```
sudo docker login
```
*# Push to DockerHub
```
sudo docker push papazeus42/piep
```
