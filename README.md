# Deploying Spark on Kubernetes with EC2 cluster
Code in the repo allows you to create a cluster out of one or more AWS EC2 instance(s), then create Apache Spark master and worker pods and run your workflows on the top of it. 

<br/>

## Prerequisits
- At least one AWS EC2 instance (recommended at least 2 CUP, 4 GB each).
- EC2s should run in the same subnet (or should be able to communicate).
- Inbound Security Group rules:
  - Open all<sup>*</sup> TCP traffic on `port 22` to enable SSH into EC2 instances.
  - Open all<sup>*</sup> TCP traffic incoming from your Security Group (so that your EC2 can communicate).
  - Optional: open all<sup>*</sup> TCP traffic on `port 30007` in order to access Spark WebUI (see section 2.2).

<sup>*</sup> Alternative only your own IP can be whitelisted instead of all traffic (`0.0.0.0/0`).

<br/>


## 1. Setting the K8s EC2 cluster.

## Setup
In order to ensure compatibility over time, specific versions of components were used - they can be edited to `latest`/`master` to suit your needs.
### 1.1 SSH into EC2 instance (on all EC2 instances):
```sh
ssh -i <public-key>.pem ubuntu@<ec2-ip>
```
### 1.2 Setup env vars (on all EC2 instances):
```sh
echo "UBUNTU_VERSION: 22.04 LTS" 
echo "SPARK_VERSION: 3.1.2"
echo "HADOOP_VERSION: 3.3.1"

export DOCKER_CE_VERSION="5:20.10.17~3-0~ubuntu-jammy"
export K8S_VERSION="1.23.0-00"
export FLANNEL_CNI="v0.19.0"
```

### 1.3 Install [Docker Engine](https://docs.docker.com/engine/install/ubuntu/) (on all EC2 instances):
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

### 1.4 Install [Kubernetes (kubelet, kubeadm, kubectl)](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) (on all EC2 instances):
```sh
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet=$K8S_VERSION kubeadm=$K8S_VERSION kubectl=$K8S_VERSION
```


### 1.5 Enable the use of iptables (on all EC2 instances):
```sh
echo "net.bridge.bridge-nf-call-iptables=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### 1.6 Initiate the cluster (only on K8s Master EC2 instance):
```sh
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
Output should be the cluster join command - let's save it! It should looke like:
```sh
kubeadm join 123.10.10.01:6443 --token 123asd.13asda34afa \
    --discovery-token-ca-cert-hash sha256:21cxvz3vcx123zcvxzvcx2z13cvzvc123123cv12zx3vc123vc12z3vc1zcv23
```
Next:
```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 1.7 Deploy [Flannel CNI](https://github.com/flannel-io/flannel) (only on K8s Master EC2 instance):
```sh 
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/${FLANNEL_CNI}/Documentation/kube-flannel.yml
```
### 1.7 Join other EC2 node(s) into K8s cluster (on all K8s Worker EC2 instances):
```sh
# Paste and run the kubeadm join link
kubeadm join 123.10.10.01:6443 --token 123asd.13asda34afa \
    --discovery-token-ca-cert-hash sha256:21cxvz3vcx123zcvxzvcx2z13cvzvc123123cv12zx3vc123vc12z3vc1zcv23
```
### 1.8 Confirm on the master node:
```sh
kubectl get nodes
```
### 💥💥💥 - our cluster has grown!

________________

<br/>

## 2. Deploying the Spark App.

### 2.1 Copy YAML files into your K8s Master EC2 instance:
```sh
# Option 1 - clone the repo (execute from EC2 instance):
sudo apt-get update
sudo apt-get install git
git clone https://github.com/mr-sailor/k8s-spark-on-ec2.git

# Option 2 - copy files using scp (execute from your local workstation):
scp -i /path/to/your/key/<public-key>.pem -r ./folder ubuntu@<ec2-ip>:/home/ubuntu/folder
``` 

### 2.2 Deploy Spark master:
```sh
# Starts Spark master pod
kubectl apply -f spark-master-deployment.yaml
# Opens Spark master on ports 8080 & 7077 (within cluster only)
kubectl apply -f spark-master-service.yaml
# Opens Spark master to the world - you can access Spark WebUI under <node-ip-where-spark-master-is-hosted>:30007
# Do not deploy node port if you are running any production workflows!
kubectl apply -f spark-master-webui-node-port.yaml
```
### 2.3 Deploy Spark workers:
```sh
kubectl apply -f spark-worker-deployment.yaml
```
### 2.3 Test Spark Web Ui:
You can go to `http://<node-ip-where-spark-master-is-hosted>:30007` and you should be able to see Spark Web UI.

### 2.4 Test Spark images
```sh
kubectl get pods 
kubectl exec -it spark-master-1231234fsdf-098dfs pyspark
```

<br/>

## 3. Building your own image.

<br/>

# Creds
Spark image and deployments were inspired by [testdrivenio](https://github.com/testdrivenio/spark-kubernetes).