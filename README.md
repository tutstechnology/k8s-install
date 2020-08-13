# Install Cluster K8S 
## (Master and Workers) and services: Docker, Dashboard, Metrics, Ambassador and Prometheus

-----------------------
### Information:

_This manual summarizes the good installation practices for the Kubernetes Cluster. It is the combination of several technical references that I found on the internet and I gathered them all in this manual._

_The installation simulation is based on a Master server and two Workers server nodes. For each step, there is information (master or worker) indicating where the configuration and deployment should be performed._


**In this installation plan, we will use the following servers / ip:**

`aws-prd-k8smaster01`     `10.130.200.25`   

`aws-prd-k8sworker01`     `10.130.200.26`   

`aws-prd-k8sworker02`     `10.130.200.27`   



-----------------------
### Port requirements for cluster services:

| service | master | worker | port type |
|:-:|:-:|:-:|:-:|
| kubelet  | 43925*  | 39305*  | dynamic  |
| kubelet  | 10250  | 10250  | fixed  |
| kubelet  | 10248  | 10248  | fixed |
| container  | 42835*  | 41425*  | dynamic |
| kube-sche  | 10259  |   | fixed  |
| kube-apis  | 6443  |   | fixed  |
| kube-cont  | 10257  |   | fixed  |
| kube-prox  | 10256  | 10256  | fixed  |
| kube-prox  | 10249  | 10249  | fixed  |
| kubectl  | 5100  |   | fixed  |

###### _*Example ports_

_This information is important for defining firewall rules between nodes._

###### After completing the installation using all the steps below, it is possible to consult the ports used. Run the command:
```
lsof -i -P -n | grep LISTEN
```

-----------------------
### Installation index:

**step 1 to step 5 -** _Requirements and settings for Linux - used (Ubuntu 20.04)._

**step 6            -** _Docker CE installation._

**step 7 to step 12 -** _Kuberntes installation._

**step 13 to step 17 -** _Dashboard installation._

**step 18 to step 19 -** _Metrics installation._

**step 20 to step 21 -** _Ambassador installation._

**step 22 to END -** _Prometheus (Building step by step)._

-----------------------


### Config / Linux Requirements:


#### **Step 1**: | Master | Worker | - _Update the latest packages using the apt `update command`:_
```
sudo apt update
```

#### **Step 2**: | Master | Worker | - _Define the name and ip of the servers that form the k8s cluster. Edit the `/etc/hosts` file and add the cluster servers:_
Example:
```
127.0.0.1 localhost
10.130.200.25   aws-prd-k8smaster01
10.130.200.26   aws-prd-k8sworker01
10.130.200.27   aws-prd-k8sworker02
```

#### **Step 3**: | Master | Worker | - _Set the `hostname` for the Master server:_
Example:
```
hostnamectl set-hostname "server name"
```

#### **Step 4**: | Master | Worker | - _Disable memory `SWAP`:_
```
sudo swapon -s
sudo swapoff -a
```

Comment if you have the Swap line on `fstab`:
```
sudo vim /etc/fstab
```
Exemple: **#**/dev/mapper/hakase--labs--vg-swap_1 none            swap    sw              0       0


#### **Step 5**: | Master | Worker | - _Restart the server:_
```
sudo reboot
```
-----------------------
### Install Docker CE:

#### **Step 6**: | Master | Worker | - _Installing Docker:_
```
sudo apt update
```

```
sudo apt install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```

```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

```
sudo apt-key fingerprint 0EBFCD88
```
**The result of the above command should be this:**

_`pub   rsa4096 2017-02-22 [SCEA]`_

_`9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88`_
      
_`uid           [ unknown] Docker Release (CE deb) <docker@docker.com>`_

_`sub   rsa4096 2017-02-22 [S]`_


```
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

```
sudo apt update
```

```
sudo apt install docker-ce docker-ce-cli containerd.io
```

```
apt-cache madison docker-ce
```
**Result:**

_`docker-ce | 5:19.03.12~3-0~ubuntu-focal | https://download.docker.com/linux/ubuntu focal/stable amd64 Packages`_
 
_`docker-ce | 5:19.03.11~3-0~ubuntu-focal | https://download.docker.com/linux/ubuntu focal/stable amd64 Packages`_
 
`docker-ce | 5:19.03.10~3-0~ubuntu-focal | https://download.docker.com/linux/ubuntu focal/stable amd64 Packages`
 
`docker-ce | 5:19.03.9~3-0~ubuntu-focal | https://download.docker.com/linux/ubuntu focal/stable amd64 Packages`_

**IMPORTANT:** 
Replace `<VERSION_STRING>` and `<VERSION_STRING>` in the command below

sudo apt-get install docker-ce=`<VERSION_STRING>` docker-ce-cli=`<VERSION_STRING>` containerd.io

**Example:**
```
apt-get install docker-ce=5:19.03.12~3-0~ubuntu-focal docker-ce-cli=5:19.03.12~3-0~ubuntu-focal containerd.io
```

```
sudo docker run hello-world
```

```
docker -v
```

-----------------------
### Installation and configuration Kubernetes:

#### **Step 7**: | Master | Worker | - _Install Kubernetes Packages (`kubeadm`, `kubelet`, `kubectl`):_
```
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubeadm kubelet kubectl kubernetes-cni
```

#### **Step 8**: | Master | - _Cluster K8S configuration. Definition of the invite and pod network:_
```
kubeadm init --apiserver-advertise-address $(hostname -i) --pod-network-cidr=10.244.0.0/16
```
**IMPORTANT:** The command above will start the cluster and then display the command line I need to execute on my other nodes.
NOTE the information to join the Workers servers. 
Command result example:
```
kubeadm join 10.130.200.25:6443 --token kqbyqy.q6543jyyx6xl84yd \
    --discovery-token-ca-cert-hash sha256:94a749272471966abeb39c7bb74a597603994e091cfda17a1915b1eb72625c2c
```

#### **Step 9**: | Master | - _Cluster K8S configuration:_
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### **Step 10**: | Master | - _Cluster K8S configuration. Definition of the `network component` and driver for Kuberntes:_

**IMPORTANT:** Choose only one of the models below:

Model 1 - Flannel
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

Model 2 - WeaveWorks
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

List podsnetwork:
```
kubectl get pods -n kube-system
```

#### **Step 11**: | Worker | - _Join Worker node to the Cluster:_

**IMPORTANT:** Use the command output seen in **step 8**
```
kubeadm join 10.130.200.25:6443 --token kqbyqy.q6543jyyx6xl84yd \
    --discovery-token-ca-cert-hash sha256:94a749272471966abeb39c7bb74a597603994e091cfda17a1915b1eb72625c2c
```

#### **Step 12**: | Master | - _Commands to check the nodes/pods:_
```
kubectl get nodes
kubectl get pods --all-namespaces
```
-----------------------
### Installation and configuration Dashboard (v2.0.3):

#### **Step 13**: | Master | - _Dashboard Deploy:_

Oficial Repository:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.3/aio/deploy/recommended.yaml
```

#### **Step 14**: | Master | - _Dashboard Deploy - Create `Admin User`:_

File creation (1) `createuseradmin-user.yaml`:
```
nano createuseradmin-user.yaml
```
Copy and paste the information below into the file:
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
```

File creation (2) `dashboard-adminuser.yaml`:
```
nano dashboard-adminuser.yaml
```
Copy and paste the information below into the file:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

Offline Deploy:
```
kubectl apply -f [local directory] ../createuseradmin-user.yaml
kubectl apply -f [local directory] ../dashboard-adminuser.yaml
```

#### **Step 15**: | Master | - _Dashboard Deploy - Getting a Bearer `Token`:_

**IMPORTANT:** The command below will generate a token, make a NOTE to use in the console login.

```
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

Command result example:
```
Name:         admin-user-token-j8wqq
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: 7e03d604-e4f7-4925-85dc-cbd9feb1ff51

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  20 bytes
token:    
eyJhbGciOiJSUzI1NiIsImtpZCI6ImlKeTdwWHhyMnNUOHZ3MVpuN0JqRnVQUWVSajNEOUM4bUtyNnRMNXBGZGMifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLWo4d3FxIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI3ZTAzZDYwNC1lNGY3LTQ5MjUtODVkYy1jYmQ5ZmViMWZmNTEiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.WVWY9SQAqHaiNF8yoJnpjVmXIBBPbkxksrwwh30eTq370gCObR5hsCmFBG60zPUgfehKZ9IFFFAkgcB0m2JAid8IAN1MaQQRuTp1VYg7z-xjqvC1U9xh3t-7k519EVq9AER_VtDiqh0GRVHRw5qa8jLG7ObYua6PyZn_cz6VvSOTT3bsK_GQHa5UCCW9P2suyZm4B4ztdtjCvpN7kQBFo5Mf9EmxCV5lsKI5EpBthRpniPItP-haKLyW14acJK3Mocuwg09cVOwcKeMaqhoMLNEIyn35o3EOTuG6xDVQFqVsnfateiRaN_Wza63cQ8UZtWrhfFi5MxgpwbfnHyHPUA
root@aws-prd-k8smaster01:/etc/kubernetes/manifests/tutstechnology/dashboard/v203#
```

Below is the token that should be noted:

`eyJhbGciOiJSUzI1NiIsImtpZCI6ImlKeTdwWHhyMnNUOHZ3MVpuN0JqRnVQUWVSajNEOUM4bUtyNnRMNXBGZGMifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLWo4d3FxIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI3ZTAzZDYwNC1lNGY3LTQ5MjUtODVkYy1jYmQ5ZmViMWZmNTEiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.WVWY9SQAqHaiNF8yoJnpjVmXIBBPbkxksrwwh30eTq370gCObR5hsCmFBG60zPUgfehKZ9IFFFAkgcB0m2JAid8IAN1MaQQRuTp1VYg7z-xjqvC1U9xh3t-7k519EVq9AER_VtDiqh0GRVHRw5qa8jLG7ObYua6PyZn_cz6VvSOTT3bsK_GQHa5UCCW9P2suyZm4B4ztdtjCvpN7kQBFo5Mf9EmxCV5lsKI5EpBthRpniPItP-haKLyW14acJK3Mocuwg09cVOwcKeMaqhoMLNEIyn35o3EOTuG6xDVQFqVsnfateiRaN_Wza63cQ8UZtWrhfFi5MxgpwbfnHyHPUA`
 


#### **Step 16**: | Master | - _Dashboard - Use Proxy:_
```
kubectl proxy --address='0.0.0.0' --port=5100 --accept-hosts='.*' &
```

**IMPORTANT:** To access the console of a remote computer, use the command to route ports through SSH. See how in the next step.


#### **Step 17**: | Client | - _Dashboard - Console access:_

_Use this command on the customer's computer, which will access a web console (
Example for server ip `10.130.200.25`):_

```
ssh -L 5888:127.0.0.1:5100 10.130.200.25
```
_or (with Access Key):_
```
ssh -i mykey.pem -L 5888:127.0.0.1:5100 ubuntu@10.130.200.25
```

Now, access the console using the following address:
```
http://localhost:5888/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```
**IMPORTANT:** Use the Token generated in step 15.


-----------------------
### Installation and configuration Metrics (v0.3.7):

#### **Step 18**: | Master |- _Metrics Deploy:_

Oficial Repository:
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.7/components.yaml
```

```
kubectl get apiservices |egrep metrics
kubectl get deploy,svc -n kube-system |egrep metrics-server
```


#### **Step 19**: | Master | - Metrics Deploy - _Modify Parameters:_


**IMPORTANT:** Depending on your cluster setup, you may also need to change flags passed to the Metrics Server container. Most useful flags:

Use:

_**--kubelet-insecure-tls**      o not verify CA of serving certificates presented by Kubelets.  For testing purposes only.
**--kubelet-preferred-address-types strings**        The priority of node address types to use when determining which address to use to connect to a particular node (default [Hostname,InternalDNS,InternalIP,ExternalDNS,ExternalIP])_

```
kubectl -n kube-system edit deploy metrics-server
```
_You must include the following lines in the file.:_
```
       command:
       - /metrics-server
       - --kubelet-insecure-tls
       - --kubelet-preferred-address-types=InternalIP
```

Example:
_insert after line 157_
```
...
spec:
     containers:
     - args:
       - --cert-dir=/tmp
       - --secure-port=4443
       image: k8s.gcr.io/metrics-server/metrics-server:v0.3.7
       command:
       - /metrics-server
       - --kubelet-insecure-tls
       - --kubelet-preferred-address-types=InternalIP
       imagePullPolicy: IfNotPresent
       name: metrics-server
       ports:
       - containerPort: 4443
         name: main-port
         protocol: TCP
        resources: {}
...
```

To verify that the changes were made to the file, run the command:
```
kubectl get deploy metrics-server -n kube-system -o yaml | grep command -A 4
```

Need to restart the `kubelet` service:
```
systemctl restart kubelet
systemctl enable kubelet
```

After this modification, the `kubectl top` command can now be used without errors.

Examples:
```
kubectl top nodes
kubectl top pods --all-namespaces
kubectl top pods --all-namespaces --sort-by=memory
```
------
### Installation Ambassador Edge Stack:

#### **Step 20**: | Master | - _Ambassador Deploy:_
```
kubectl apply -f https://www.getambassador.io/yaml/aes-crds.yaml && \
kubectl wait --for condition=established --timeout=90s crd -lproduct=aes && \
kubectl apply -f https://www.getambassador.io/yaml/aes.yaml && \
kubectl -n ambassador wait --for condition=available --timeout=90s deploy -lproduct=aes
```

#### **Step 21**: | Master | - _Ambassador Deploy - Determine the IP address or hostname of your cluster:_
```
kubectl get -n ambassador service ambassador -o "go-template={{range .status.loadBalancer.ingress}}{{or .ip .hostname}}{{end}}"
```





------





