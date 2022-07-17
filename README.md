# Deploying Spark on Kubernetes with EC2 cluster

## Setting the K8s EC2 cluster
## Prerequisits
- 1
- 2
- 3

## Setup
In order to ensure of compatibility over time, specific versons of components were used - they can be edited to latest/master to suit the needs.
### 1. Setup env vars:
```sh
echo "UBUNTU_VERSION: 22.04 LTS" 
echo "SPARK_VERSION: 3.1.2"
echo "HADOOP_VERSION: 3.3.1"

export DOCKER_CE_VERSION="5:20.10.17~3-0~ubuntu-jammy"
export K8S_VERSION="1.23.0-00"
export FLANNEL_CNI="v0.19.0"
```

### 2.Install [Docker Engine](https://docs.docker.com/engine/install/ubuntu/):
```sh
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
apt-cache madison docker-ce

sudo apt-get install -y docker-ce=$DOCKER_CE_VERSION
```

### 3.Install [Kubernetes (kubelet, kubeadm, kubectl)](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/):
```sh
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet=$K8S_VERSION kubeadm=$K8S_VERSION kubectl=$K8S_VERSION
```


### 4.Enable the use of iptables:
```sh
echo "net.bridge.bridge-nf-call-iptables=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### 5. On Master node only!
```sh
# initiate the cluster
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
We should see cluster join command - let's save it! It should looke like:
```sh
kubeadm join 123.10.10.01:6443 --token 123asd.13asda34afa \
    --discovery-token-ca-cert-hash sha256:21cxvz3vcx123zcvxzvcx2z13cvzvc123123cv12zx3vc123vc12z3vc1zcv23
```
Next:
```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Initiate Flannel CNI
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/${FLANNEL_CNI}/Documentation/kube-flannel.yml
```
### 6. On other EC2 node(s):
```sh
# Paste and run the kubeadm join link
kubeadm join 123.10.10.01:6443 --token 123asd.13asda34afa \
    --discovery-token-ca-cert-hash sha256:21cxvz3vcx123zcvxzvcx2z13cvzvc123123cv12zx3vc123vc12z3vc1zcv23
```
### 7. Confirm on the master node:
```sh
kubectl get nodes
```
## 💥💥💥 - our cluster has grown!

mkdir code

## The App
