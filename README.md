# I have setup to setting up a kubernetes cluster using Kubeadm with one master and one worker nodes.

### Clone this repository  https://github.com/praveensivakumar1998/Setup-Kubernetes-Cluster-using-Kubeadm-and-Deploy-a-simple-application.git

Kubeadm is an excellent tool to set up a working kubernetes cluster in less time. It does all the heavy lifting in terms of setting up all kubernetes cluster components. Also, It follows all the configuration best practices for a kubernetes cluster.

### What is Kubeadm?
Kubeadm is a tool to set up a minimum viable Kubernetes cluster without much complex configuration. Also, Kubeadm makes the whole process easy by running a series of prechecks to ensure that the server has all the essential components and configs to run Kubernetes.

## Kubeadm Setup Prerequisites
### Following are the prerequisites for Kubeadm Kubernetes cluster setup.

1. Minimum two Ubuntu nodes [One master and one worker node]. You can have more worker nodes as per your requirement.
2. The master node should have a minimum of 2 vCPU and 2GB RAM. For the worker nodes, a minimum of 1vCPU and 2 GB RAM is recommended.
3. 10.X.X.X/X network range with static IPs for master and worker nodes. We will be using the 192.x.x.x series as the pod network range that will be used by the Calico network plugin. Make sure the Node IP range and pod IP range don’t overlap.
   
   ![image](https://github.com/praveensivakumar1998/Setup-Kubernetes-Cluster-using-Kubeadm-and-Deploy-a-simple-application/assets/108512714/3a8f9a0f-a653-4bd8-9743-adaaafe603ca)


## Kubernetes Cluster Setup Using Kubeadm

1. Install container runtime on all nodes- We will be using docker runtime.
2. Install Kubeadm, Kubelet, and kubectl on all the nodes.
3. Initiate Kubeadm control plane configuration on the master node.
4. Save the node join command with the token.
5. Install the Calico network plugin (operator).
6. Join the worker node to the master node (control plane) using the join command.
7. Validate all cluster components and nodes.

## Step 1: Enable iptables Bridged Traffic on all the Nodes   

Execute the following commands on all the nodes for IPtables to see bridged traffic. Here we are tweaking some kernel parameters and setting them using sysctl.
```
sudo apt update -y
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

## Step 2: Disable swap on all the Nodes
For kubeadm to work properly, you need to disable swap on all the nodes using the following command.

```
sudo swapoff -a
(crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true
```

## Step 3: Install Docker runtime on all nodes

```
sudo su 
apt install -y docker.io
usermod -aG docker ubuntu
```

## Step 4: Install Kubeadm & Kubelet & Kubectl on all Nodes

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
Download the GPG key for the Kubernetes APT repository on all the nodes.

```
sudo mkdir -p /etc/apt/keyrings

sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get install -y kubelet kubeadm kubectl

#Add the node IP to KUBELET_EXTRA_ARGS

sudo apt-get install -y jq
local_ip="$(ip --json addr show ens5 | jq -r '.[0].addr_info[] | select(.family == "inet") | .local')"
cat > /etc/default/kubelet << EOF
KUBELET_EXTRA_ARGS=--node-ip=$local_ip
EOF
```

## Step 5: Initialize Kubeadm On Master Node To Setup Control Plane

### Master Node with Private IP: If you have nodes with only private IP addresses the API server would be accessed over the private IP of the master node.

```
IPADDR="10.0.0.10"
NODENAME=$(hostname -s)
POD_CIDR="192.168.0.0/16
```

Now, initialize kubeadm in master node

```
sudo kubeadm init --apiserver-advertise-address=$IPADDR  --apiserver-cert-extra-sans=$IPADDR  --pod-network-cidr=$POD_CIDR --node-name $NODENAME --ignore-preflight-errors Swap
```
On a successful kubeadm initialization, you should get an output with kubeconfig file location and the join command with the token as shown below. Copy that and save it to the file. we will need it for joining the worker node to the master.

![init](https://github.com/praveensivakumar1998/Setup-Kubernetes-Cluster-using-Kubeadm-and-Deploy-a-simple-application/assets/108512714/f6358b36-deb7-4d97-99f3-7d72466f46b3)

Use the following commands from the output to create the kubeconfig in master so that you can use kubectl to interact with cluster API.
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Now, verify the kubeconfig by executing the following kubectl command to list all the pods in the kube-system namespace.
```
kubectl get po -n kube-system
```
You should see the following output. You will see the two Coredns pods in a pending state. It is the expected behavior. Once we install the network plugin, it will be in a running state.
![image](https://github.com/praveensivakumar1998/Setup-Kubernetes-Cluster-using-Kubeadm-and-Deploy-a-simple-application/assets/108512714/8ca3d056-83d8-4e8f-b09b-f4abf12f65d6)

You verify all the cluster component health statuses using the following command.
```
kubectl get --raw='/readyz?verbose'
```
![image](https://github.com/praveensivakumar1998/Setup-Kubernetes-Cluster-using-Kubeadm-and-Deploy-a-simple-application/assets/108512714/0b3956ed-6f3f-4231-89cb-092fea253322)

You can get the cluster info using the following command.

```
kubectl cluster-info
```
![image](https://github.com/praveensivakumar1998/Setup-Kubernetes-Cluster-using-Kubeadm-and-Deploy-a-simple-application/assets/108512714/5ad56e22-d69e-425e-b0ed-fb03029cdb5c)

By default, apps won’t get scheduled on the master node. If you want to use the master node for scheduling apps, taint the master node.
```
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```
## Step 6: Join Worker Nodes To Kubernetes Master Node

Now, let’s join the worker node to the master node using the Kubeadm join command you have got in the output while setting up the master node.

```Note: use this below join command in worker node```

```
kubeadm join 10.1.10.118:6443 --token d2ru3h.4xwgikbakeiyywjd \
        --discovery-token-ca-cert-hash sha256:2850c930b8839cd14435a251a263081787564281d30d19fd3f7d120c26826f9b
```
![wn](https://github.com/praveensivakumar1998/Setup-Kubernetes-Cluster-using-Kubeadm-and-Deploy-a-simple-application/assets/108512714/0e585492-c92c-4806-b1fc-783534fa2fae)

Now execute the kubectl command from the master node to check if the node is added to the master.

```
kubectl get nodes
```
### Example output,
```
NAME             STATUS     ROLES           AGE     VERSION
ip-10-1-10-118   NotReady   control-plane   8m50s   v1.29.3
ip-10-1-10-234   NotReady   <none>          75s     v1.29.3
```
In the above command, the ROLE is <none> for the worker nodes. You can add a label to the worker node using the following command
```
kubectl label node ip-10-1-10-234  node-role.kubernetes.io/worker=worker
```
Check the node name would renamed by label command
```
kubectl get nodes
```
### Example output,
```
ip-10-1-10-118   NotReady   control-plane   10m     v1.29.3
ip-10-1-10-234   NotReady   worker          2m43s   v1.29.3
```
## Step 7: Install Calico Network Plugin for Pod Networking
Kubeadm does not configure any network plugin. You need to install a network plugin of your choice for kubernetes pod networking and enable network policy.
I am using the Calico network plugin for this setup.
```
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```
Note: make sure to apply a network plugin in master node
![image](https://github.com/praveensivakumar1998/Setup-Kubernetes-Cluster-using-Kubeadm-and-Deploy-a-simple-application/assets/108512714/bebc2d6c-856a-4e44-bea5-a2904251716c)

After a couple of minutes, if you check the pods in kube-system namespace, you will see calico pods and running CoreDNS pods.
```
kubectl get po -n kube-system
```
![image](https://github.com/praveensivakumar1998/Setup-Kubernetes-Cluster-using-Kubeadm-and-Deploy-a-simple-application/assets/108512714/ec04c1e9-27fd-45f2-bde4-d35b2f1a951d)

## Step 8: Run simple nginx application in kubernetes cluster 
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 32000
```
Run Nginx Application using apply cmd 
```
kubectl apply -f nginx-app.yaml
```
### Execution output:
```
deployment.apps/nginx-deployment created
service/nginx-service created
```
Check the application is running as pods
```
kubectl get pods
```
### Example output:
```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7c79c4bf97-l2n5n   1/1     Running   0          36s
nginx-deployment-7c79c4bf97-zqgrd   1/1     Running   0          36s
```
let's check the nginx application running in a worker node
```http://workernode-publicip:32000```
![image](https://github.com/praveensivakumar1998/Setup-Kubernetes-Cluster-using-Kubeadm-and-Deploy-a-simple-application/assets/108512714/fb85c6f4-fcc4-4e92-883e-c01748c93c35)


you can also check the masternode by masternode public ip with port 32000










