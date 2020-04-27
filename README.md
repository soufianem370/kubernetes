# kubernetes
Kubernetes playground

# installation metalLB
On Cloud infrastructure, the provider ensures laodBalancing role for your k8s cluster, but on premise cluster needs a deployment of  Laod balancing solution which allows to you to use laodbalancer function, MetalLB is one of the quintessential solution which permit to do this:

1) telecharger manifest depuis le site officiel
Réf: https://metallb.universe.tf/installation/

kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.8.3/manifests/metallb.yaml

2- Add a configMap to define a range of ExternalIP-addresses
run kubectl get nodes -o wide to know about the range of addresses you need to reserve.

```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: k8s-master-ip-space
      protocol: layer2
      addresses:
      - 172.23.0.100-172.23.0.250    
```

# install and configure ingress controller with nginx Bare Metal

## install nginx controller
ressources:

namespace
service account
cluster Role
cluster Role Binding
ConfigMap
Secret
Daemonset

1)copy repo from github


```bash
▶ git clone https://github.com/nginxinc/kubernetes-ingress.git
▶ cd kubernetes-ingress
▶ cd deployments

```
## 1. Create a Namespace, a SA, the Default Secret, the Customization Config Map, and Custom Resource Definitions

1. Create a namespace and a service account for the Ingress controller:
    ```
    kubectl apply -f common/ns-and-sa.yaml
    ```

1. Create a secret with a TLS certificate and a key for the default server in NGINX:
    ```
    $ kubectl apply -f common/default-server-secret.yaml
    ```

    **Note**: The default server returns the Not Found page with the 404 status code for all requests for domains for which there are no Ingress rules defined. For testing purposes we include a self-signed certificate and key that we generated. However, we recommend that you use your own certificate and key.

1. Create a config map for customizing NGINX configuration (read more about customization [here](configmap-and-annotations.md)):

    ```
    $ kubectl apply -f common/nginx-config.yaml
    ```

1. (Optional) To use the [VirtualServer and VirtualServerRoute](virtualserver-and-virtualserverroute.md) resources, create the corresponding resource definitions:
    ```
    $ kubectl apply -f common/custom-resource-definitions.yaml
    ```
    Note: in Step 3, make sure the Ingress controller starts with the `-enable-custom-resources` [command-line argument](cli-arguments.md).

## 2. Configure RBAC

If RBAC is enabled in your cluster, create a cluster role and bind it to the service account, created in Step 1:
```
$ kubectl apply -f rbac/rbac.yaml
```

**Note**: To perform this step you must be a cluster admin. Follow the documentation of your Kubernetes platform to configure the admin access. For GKE, see the [Role-Based Access Control](https://cloud.google.com/kubernetes-engine/docs/how-to/role-based-access-control) doc.

## 3. Deploy the Ingress Controller

We include two options for deploying the Ingress controller:
* *Deployment*. Use a Deployment if you plan to dynamically change the number of Ingress controller replicas.
* *DaemonSet*. Use a DaemonSet for deploying the Ingress controller on every node or a subset of nodes.

### 3.1 Create a DaemonSet

For NGINX, run:
```
$ kubectl apply -f daemon-set/nginx-ingress.yaml
```
Kubernetes will create an Ingress controller pod on every node of the cluster. Read [this doc](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) to learn how to run the Ingress controller on a subset of nodes, instead of every node of the cluster.

### 3.2 Check that the Ingress Controller is Running

Run the following command to make sure that the Ingress controller pods are running:
```
$ kubectl get pods --namespace=nginx-ingress
```
## 4. Deploy pods for testing your ingress Controller

### first deployment with home page

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: nginx
  name: nginx-deploy-main
spec:
  replicas: 1
  selector:
    matchLabels:
      run: nginx-main
  template:
    metadata:
      labels:
        run: nginx-main
    spec:
      containers:
      - image: nginx
        name: nginx
```


### second deployment an pod with page green

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: nginx
  name: nginx-deploy-green
spec:
  replicas: 1
  selector:
    matchLabels:
      run: nginx-green
  template:
    metadata:
      labels:
        run: nginx-green
    spec:
      volumes:
      - name: webdata
        emptyDir: {}
      initContainers:
      - name: web-content
        image: busybox
        volumeMounts:
        - name: webdata
          mountPath: "/webdata"
        command: ["/bin/sh", "-c", 'echo "<h1>I am <font color=green>GREEN</font></h1>" > /webdata/index.html']
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: webdata
          mountPath: "/usr/share/nginx/html"
```

### Third deployment an pod with page blue

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: nginx
  name: nginx-deploy-blue
spec:
  replicas: 1
  selector:
    matchLabels:
      run: nginx-blue
  template:
    metadata:
      labels:
        run: nginx-blue
    spec:
      volumes:
      - name: webdata
        emptyDir: {}
      initContainers:
      - name: web-content
        image: busybox
        volumeMounts:
        - name: webdata
          mountPath: "/webdata"
        command: ["/bin/sh", "-c", 'echo "<h1>I am <font color=blue>BLUE</font></h1>" > /webdata/index.html']
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: webdata
          mountPath: "/usr/share/nginx/html"
```

### 5. Create service to expose yours pods (by default clusterIP if you don't specify the type)

```
kubectl expose deploy nginx-deploy-main --port 80
kubectl expose deploy nginx-deploy-blue --port 80
kubectl expose deploy nginx-deploy-green --port 80
```

### 6.add ingress ressource

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-resource-2
spec:
  rules:
  - host: nginx.example.com
    http:
      paths:
      - backend:
          serviceName: nginx-deploy-main
          servicePort: 80
  - host: blue.nginx.example.com
    http:
      paths:
      - backend:
          serviceName: nginx-deploy-blue
          servicePort: 80
  - host: green.nginx.example.com
    http:
      paths:
      - backend:
          serviceName: nginx-deploy-green
          servicePort: 80
```

# install storageClass nfs avec une helm nfs-client-provisioner

1)installer nfs server voir la doc https://github.com/soufianem370/admin_linux

2)create partage sur le serveur nfs /srv/nfs/kubedata

3)dans chaque serveur du cluster k8s installé nfs-utils

```
yum install nfs-utils
```
4)sur le cluster kubernetes lancer la creation d'une storageclass avec la charte helm

```
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm repo update
helm install stable/nfs-client-provisioner --set nfs.server=172.23.0.1 --set nfs.path=/srv/nfs/kubedata
```
nfs.server=l'adresse du serveur phisique nfs
nfs.path=le dossier partagé ou export nfs

5)mettre le storage classe créer entant que storageclasse par-defaut

```
kubectl patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}' 
```

# installation prometheus avec grafana en utilisant helm
install helm in your cluster
install chart prometheus
```
helm inspect stable/prometheus > /tmp/prom.values
```
change type off service on NodePort
vim /tmp/prom.values
```
    externalIPs: []

    loadBalancerIP: ""
    loadBalancerSourceRanges: []
    servicePort: 80
    nodePort: 31114
    sessionAffinity: None
    type: NodePort
```
```
helm install --values /tmp/prom.values --name myprom1 stable/prometheus
```
or 

```
helm install --name myprom1 --set server.service.type=NodePort --set server.service.nodePort=32222  stable/prometheus 
```

### si votre service n'est pas créer en mode NodePort il faut supprimer le service et le recrier manuellement

```
 #kubectl delete svc myprom1-prometheus-server
 #kubectl expose deployment myprom1-prometheus-server --type=NodePort --port=80 --target-port=9090
```
## installé grafana 
1) methode :
```
helm install stable/grafana --name mygrafana --set persistence.enabled=true --set persistence.accessModes={ReadWriteOnce} --set persistence.size=8Gi --set service.type=NodePort --set service.nodePort=32221
```
user: admin 
password nous pouvons le récuperé depuis le secret du chart

```
kubectl get secret --namespace default mygrafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```
mygrafana: nom du chart

2) méthode
helm inspect stable/grafana > /tmp/grafana.values

vim /tmp/grafana.values
```
service:
  type: NodePort
  port: 80
  nodePort: 32211
  targetPort: 3000
    # targetPort: 4181 To be used with a proxy extraContainer
  annotations: {}
  labels: {}
  portName: service
```
```
kubectl expose deployment grafana1 --type=NodePort --port=80 --target-port=3000
```
user: admin 
password nous pouvons le récuperé depuis le secret du chart

```
kubectl get secret --namespace default mygrafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```
mygrafana: nom du chart

## install rancher on your cluster version is compatible with prometheus

1)run container runcher on your local machine this version is compatible with prometheus

```
docker run --name rancher -d -p 80:80 -p 443:443 -v rancher-data:/var/lib/rancher rancher/rancher:v2.3.5```
2)execute this manifests on your cluster k8S
```
curl --insecure -sfL https://localhost/v3/import/qv476x7mtw2kw2fp6fz452glcmknsgwj6cvx7jqxt9mrv2qts2mwdd.yaml | kubectl apply -f -
```
## installation cluster with kubeadm
==>tous ces taches à executer sur l'ensemble des machines worker et master

```
cat <<EOF>> /etc/hosts
10.128.0.27 master-node
10.128.0.29 node-1 worker-node-1
10.128.0.30 node-2 worker-node-2
EOF
```
```
setenforce 0
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
reboot
```
```
firewall-cmd --permanent --add-port=6443/tcp
firewall-cmd --permanent --add-port=2379-2380/tcp
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --permanent --add-port=10251/tcp
firewall-cmd --permanent --add-port=10252/tcp
firewall-cmd --permanent --add-port=10255/tcp
firewall-cmd --reload
modprobe br_netfilter
echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
```
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```
```
yum install kubeadm docker -y 
systemctl enable kubelet && systemctl enable docker
systemctl start kubelet && systemctl start docker
swapoff -a
```
edit /etc/fstab to make a swap disabled permanently

==> sur le neud master intier le cluster
```
swapoff -a
kubeadm init
```
apres l'execution de cette commande nous aurons le token pour joingné les workers
```
kubeadm join 172.17.0.10:6443 --token jlsu5g.1wp4m5i3aza82yjf \
    --discovery-token-ca-cert-hash sha256:f246b98d354a5371dad50153d751b36f59f4866c0d4ce4f6b8bda8167cb49e71
```
==> sur chaque machine worker executer cette commande pour joigné le worker au cluster
```
worker# kubeadm join 172.17.0.10:6443 --token jlsu5g.1wp4m5i3aza82yjf \
    --discovery-token-ca-cert-hash sha256:f246b98d354a5371dad50153d751b36f59f4866c0d4ce4f6b8bda8167cb49e71
```

==> Installer bash-completion

```
#yum install bash-completion

#source /usr/share/bash-completion/bash_completion
```

==>Activer l’auto-complétion de kubectl

```
#echo 'source <(kubectl completion bash)' >>~/.bashrc
#kubectl completion bash >/etc/bash_completion.d/kubectl
```

## maintenance d'un cluster kubernetes kubeadm
1)sur le master
pour désinstaller tous les composants k8s
```
#kubeadm reset all
```
```
kubectl delete node <node-name>
```
pour reinstaller a nouveau votre master
il va te générer le tocken pour faire rejoindre les workers
```
#kubeadm init
.
.
.

kubeadm join 192.168.1.247:6443 --token 0g9j4w.fzc8nc7jhmkvttiv \
    --discovery-token-ca-cert-hash sha256:34ee91bf998850c90c88bb49206a9b2758441d427fc47d50cc4157f9d4a7e5d6 
```
==> si tu as oublier le token master

```
#kubeadm token generate

naoynf.c7il2s44pimqg3w0
```

==> copier le nouveau token et générer la commande join
```
#kubeadm token create naoynf.c7il2s44pimqg3w0 --print-join-command --ttl=0 
```
2) sur le worker s'il est déja joigné à un autre master il faut le disjoigné 
```
#kubeadm reset all
```
3) executé cette commande pour joigné au nouveau master

```
#kubeadm join 192.168.1.247:6443 --token 0g9j4w.fzc8nc7jhmkvttiv \
    --discovery-token-ca-cert-hash sha256:34ee91bf998850c90c88bb49206a9b2758441d427fc47d50cc4157f9d4a7e5d6
```
4) si ça marche pas vérifier la synchronisation de la date entre le worker et le master

5) regénérer le token sur le master et rejoigné le worker avec le nouveau token

6) installé un CNI calico ou weave-net 

```
kubectl apply -f https://docs.projectcalico.org/v3.9/manifests/calico.yaml
```
## install cluster K8S with kubespray

clone this repo with tag v2.11.0

```
https://github.com/kubernetes-sigs/kubespray.git
```
create inventory automatique
```
cd kybespray 
apt-get install python3-pip
pip3  install -r contrib/inventory_builder/requirements.txt
cp -rfp inventory/sample inventory/mycluster
declare -a IPS=(192.168.122.134 192.168.122.216 192.168.122.193)  #example...
CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
```



## install cluster k8s sur AWS avec Kops

1. Create Ubuntu EC2 instance
1. install AWSCLI
   ```sh 
    curl https://s3.amazonaws.com/aws-cli/awscli-bundle.zip -o awscli-bundle.zip
    apt install unzip python
    unzip awscli-bundle.zip
    #sudo apt-get install unzip - if you dont have unzip in your system
    ./awscli-bundle/install -i /usr/local/aws -b /usr/local/bin/aws
    ```
    
1. Install kubectl
   ```sh
   curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
    chmod +x ./kubectl
    sudo mv ./kubectl /usr/local/bin/kubectl
   ```
1. Create an IAM user/role  with Route53, EC2, IAM and S3 full access
1. Attach IAM role to ubuntu server

    #### Note: If you create IAM user with programmatic access then provide Access keys. 
   ```sh 
     aws configure
    ```
1. Install kops on ubuntu instance:
   ```sh
    curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
    chmod +x kops-linux-amd64
    sudo mv kops-linux-amd64 /usr/local/bin/kops
    ```
1. Create a Route53 private hosted zone (you can create Public hosted zone if you have a domain)
1. create an S3 bucket 
   ```sh
    aws s3 mb s3://dev.k8s.valaxy.in
   ```
1. Expose environment variable:
   ```sh 
    export KOPS_STATE_STORE=s3://dev.k8s.valaxy.in
   ```
1. Create sshkeys before creating cluster
   ```sh
    ssh-keygen
   ```
1. Create kubernetes cluster definitions on S3 bucket 
   ```sh 
    kops create cluster --cloud=aws --zones=ap-southeast-1b --name=dev.k8s.valaxy.in --dns-zone=valaxy.in --dns private
    ```
1. Create kubernetes cluser
    ```sh 
      kops update cluster dev.k8s.valaxy.in --yes
     ```
1. Validate your cluster 
     ```sh 
      kops validate cluster
    ```

1. To list nodes
   ```sh 
     kubectl get nodes 
   ```

#### Deploying Nginx container on Kubernetes 
1. Deploying Nginx Container
    ```sh 
      kubectl run sample-nginx --image=nginx --replicas=2 --port=80
      kubectl get pods
      kubectl get deployments
   ```
   
1. Expose the deployment as service. This will create an ELB in front of those 2 containers and allow us to publicly access them:
   ```sh 
    kubectl expose deployment sample-nginx --port=80 --type=LoadBalancer
    kubectl get services -o wide
    ```
 1. To delete cluster
    ```sh
     kops delete cluster dev.k8s.valaxy.in --yes
    ```
### kubernetes-cluster-with-autoscaling for worker-node
https://varlogdiego.com/kubernetes-cluster-with-autoscaling-on-aws-and-kops

## pour l'envirennement de test installé kind kubernetes in docker
videos (vankat): https://www.youtube.com/watch?v=4p4DqdTDqkk&t=312s
prérequise: docker installé

==> installation go
aller sur le site officiel golang
https://golang.org/dl/

```
wget https://dl.google.com/go/go1.14.1.linux-amd64.tar.gz
sudo tar zxf go1.14.1.linux-amd64.tar.gz -C /usr/local/
```

```
echo -n 'export PATH=$PATH:/usr/local/go/bin' >> ~/.zshrc
```

==> installation Kind
```
curl -Lo ./kind https://github.com/kubernetes-sigs/kind/releases/download/v0.7.0/kind-$(uname)-amd64
chmod +x ./kind
mv ./kind /home/smakhloufi/go/kind
```
```
echo -n 'export PATH=$PATH:/home/smakhloufi/go/kind' >> ~/.zshrc
```
==> créer un cluster avec un seul node
```
#kind create cluster --name kind-2 

afficher les cluster créer

#kind get clusters

pour imposter le kubeconfig
#kind get kubeconfig --name kind-2 > ~/.kube/config
```
==> créer un cluster avec un master et plusieurs worker
creer un fichier de config
```
#vim kind_config.yaml
```
copier coller cette config
```
# three node (two workers) cluster config
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```
créer le cluster
```
kind create cluster --name cluster1 --config kind_config.yaml
```

## Using Helm v3 with Kubernetes

1. Install Helm from Script

You can always to refer to other ways of installation by accessing https://helm.sh/docs/intro/install/

Helm now has an installer script that will automatically grab the latest version of Helm and install it locally.

You can fetch that script, and then execute it locally. It’s well documented so that you can read through it and understand what it is doing before you run it.
```sh
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```
if you get an error that show up run the cmd bellow to make helm executable
```sh
sudo chmod +x linux-amd64/helm && sudo mv linux-amd64/helm /usr/bin/helm
```
2. helm completion
Generate autocompletions script for the specified shell bash 
```sh
source <(helm completion bash)
```
ref: https://helm.sh/docs/helm/helm_completion/


3. Initialize a Helm Chart Repository
Once you have Helm ready, you can add a chart repository. One popular starting location is the official Helm stable charts:
```sh
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
```
to add another repository either locally or remotely:
Install the binary:
ref: https://chartmuseum.com/docs/#using-with-local-filesystem-storage
```sh
# on Linux
curl -LO https://s3.amazonaws.com/chartmuseum/release/latest/bin/linux/amd64/chartmuseum
chmod +x ./chartmuseum
mv ./chartmuseum /usr/local/bin
```
USING WITH LOCAL FILESYSTEM STORAGE
```sh
chartmuseum --debug --port=8080 --storage="local" --storage-local-rootdir="./charts-repo"&
```
then helm servecm plugin
```sh
helm plugin install https://github.com/jdolitsky/helm-servecm
```
before adding your repo try to index this directory by runing the cmd bellow: 
```sh
mkdir charts-repo && cd charts-repo/ && helm repo index .
```
finnaly add the custom repo you have been created
```sh
helm repo add local http://127.0.0.1:8080
```
to see the repo available just run the bellow cmd:
```sh
helm repo list
```
Normally you should have two repos, the default one "stable and the customized one "local"

4.Install an Example Chart
To install a chart, you can run the helm install command. Helm has several ways to find and install a chart, but the easiest is to use one of the official stable charts.
```sh
helm repo update              # Make sure we get the latest list of charts
helm search repo stable/mysql      # choose the appropriate repo to you
helm install stable/mysql --generate-name    # if you didn't mention any name for your install you must add --generate-name 
```
5. Uninstall a Release
To uninstall a release, use the helm uninstall command:
```sh
helm uninstall smiling-penguin
```
This will uninstall smiling-penguin from Kubernetes, which will remove all resources associated with the release as well as the release history.

6. Creating your first Helm Chart
to avoid creating all files and directories from scratch, use the command bellow will give you the whole chart template, make the necessary changes then apply it:
```sh
helm create my-chart
```
tree tool is a very useful managing helm charts, to highlight the files created by this cmd you can run:
```sh
tree ./my-chart
```
to install your chart use the common comd we have already used before:
```sh
helm install my-chart .
```
you can always use values.yaml file as a reference of values, it's helpful to apply your changes on it then upgrade your app already installed by runing:
```sh
helm upgrade my-chart .
```
7. How to set up a local helm Chart 
After creating your own chart, and applying all the requierements on it, absoletly you need to share this chart on work environment using the local repo or any other remote repos:
first of all copy this chart on you local repo "repostorage, then access the dir chart and package it:
```sh
mv mmy-chart ./repostorage
cd ./repostorage/my-chart && helm package .
```
update your local repo by executing:
```sh
helm repo update
```
search for your chart on that repo:
```sh
helm search repo local/
```
6. Other useful commands:
```sh
helm help
helm inspect stable/jenkins     # to get details about the app desired
helm fetch stable/jenkins      # to download the app chart
helm list                 # to get the app already installed on your k8s cluster
helm home                # to check your home dir if it contains the mandatory files
helm upgrade             # to upgrade your app after making some changes on your chart files
helm rollback my-chart 1      # to call off the last change and return back to the roolout nu 1
helm delete --purge my-chart 
```
N.B: take a look on this https://helm.sh/docs/topics/v2_v3_migration/ to know about all changes and all cmds which are outdated on helm v3, also to learn using all helm best practice. 


## Use Samba share as a Persistent storage in Kubernetes Cluster

### 1.Pre-requisites
On your Kubernetes nodes, simply install the dependency:
```bash
yum -y install cifs-utils
```
### 2.DaemonSet Installation
the recommended driver deployment method is to have a DaemonSet install the driver cluster-wide automatically.
A Docker image juliohm/kubernetes-cifs-volumedriver-intaller is available for this purpose, which can be deployed into a Kubernetes cluster using the install.yaml from this repository. The image is built FROM busybox, so the it’s essentially very small.

The installer image allows you to install without the need to compile the project. Deploying the volume driver should be as easy as bellow:
```sh
git clone https://github.com/juliohm1978/kubernetes-cifs-volumedriver.git

cd kubernetes-cifs-volumedriver

make install
```
The install target uses kubectl to create a privileged DaemonSet with pods that mount the host directory /usr/libexec/kubernetes/kubelet-plugins/volume/exec/ for installation. Check the output from the deployed containers to make sure it did not produce any errors. Crashing pods mean something went wrong.
So, once you have verified that it’s completed, the DaemonSet can be safely removed, by runing:
```sh
make delete
```
the kubelet’s default directory for volume plugins is /usr/libexec/kubernetes/kubelet-plugins/volume/exec/. This could be different if your installation changed this directory using the --volume-plugin-dir parameter.

### 3.Create a kubernetes Secret
 this secret will contain the username and the password which are used to join the cifs share:
 ```sh
kubectl  create secret generic my-secret --from-literal=username=yelmir --from-literal=password=root --type=juliohm/cifs
```
### 4. Create a Persistent Volume
The following is an example of PersistentVolume that uses the volume driver
```sh
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cifs-pv2
spec:
  capacity:
    storage: 1Gi
  flexVolume:
    driver: juliohm/cifs
    options:
      server: 192.168.43.43
      share: /partage
    secretRef:
      name: my-secret
  accessModes:
    - ReadWriteMany
```
### 5. Create the Persistent Volume Claim
The following is an example of PersistentVolumeClaim that uses the PV we created previously
```sh
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cifs-pvc2
spec:
  resources:
   requests:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
```

### 6. Use the PVC in pod
This is an examle of using PVC based on Samba storage on pods:
```sh
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
          - name: data
            mountPath: /dados
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: cifs-pv2
```
### N.B: To customize vendor name/directory
you can customize the vendor name/directory for your installation by tweaking install.yaml and defining VENDOR and DRIVER environment variables.
```sh
      containers:
        - image: juliohm/kubernetes-cifs-volumedriver-installer:2.0
          env:
            - name: VENDOR
              value: mycompany
            - name: DRIVER
              value: mycifs
```
Réf: https://k8scifsvol.juliohm.com.br/

### Velero - Backup & Restore Kubernetes Cluster

## Set up Velero tool
Download Velero v1.2.3 and copy the binary in /usr/local/bin/ :
```bash
wget https://github.com/vmware-tanzu/velero/releases/download/v1.3.2/velero-v1.3.2-linux-amd64.tar.gz
tar xfzv velero-v1.3.2-linux-amd64.tar.gz
sudo mv velero-v1.3.2-linux-amd64/velero /usr/local/bin/
```
Create credentials file (Needed for velero initialization):
```bash
cat <<EOF>> minio.credentials
[default]
aws_access_key_id=minio
aws_secret_access_key=minio123
EOF
```
Install Velero in the Kubernetes Cluster:
```bash
velero install --provider aws --plugins velero/velero-plugin-for-aws:v1.0.0 --bucket backups --provider aws --secret-file ./minio.credentials --backup-location-config region=us-east-2 --snapshot-location-config region=minio,s3ForcePathStyle=true,s3Url=http://172.17.0.66:9000 --use-restic
```
--use-restic: Velero has support for backing up and restoring Kubernetes volumes using a free open-source backup tool called restic
to explore more about this solution https://velero.io/docs/v1.3.2/restic/

Enable tab completion for preferred shell:
```bash
source <(velero completion bash)
```
## Backup commands
You can always play with --inclue/--exclude variables to get a precise backup
```bash
velero backup get
velero backup-location get
velero backup create "backup-name" --include-namespaces="namespace-name"
velero backup create "backup-name" --include-namespaces="namespace-name" --exclude-resouces=nginx-deploy
velero backup describe "backup-name"
```
## Restore commands
```bash
velero restore get
velero restore create "restore-name" --from-backup="backup-name"
velero restore describe "restore-name"
```
## How to schedule a backup
you can always use all option available on velero backup to get the k8s-objects youu would to backup
```bash
velero schedule get 
velero schedule create "schedule-name" --schedule="30 */12 * * *" --include-namespaces default 
```
N.B: if you would set up an old version such as v1.0.0 on your kubernetes environment version 1.16.0, you will be obliged to make some changes on minio/deployments (apps/v1beta1 to apps/v1 - add replicas field to spec section of deployment - add selector fields to spec section) 
 *** to set up Velero through helm repo
 Add VMware Tanzu repository to Helm repos:
```bash
 helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
```
```bash
helm install --namespace <YOUR NAMESPACE> \
--set configuration.provider=<PROVIDER NAME> \
--set-file credentials.secretContents.cloud=<FULL PATH TO FILE> \
--set configuration.backupStorageLocation.name=<PROVIDER NAME> \
--set configuration.backupStorageLocation.bucket=<BUCKET NAME> \
--set configuration.backupStorageLocation.config.region=<REGION> \
--set configuration.volumeSnapshotLocation.name=<PROVIDER NAME> \
--set configuration.volumeSnapshotLocation.config.region=<REGION> \
--set image.repository=velero/velero \
--set image.pullPolicy=IfNotPresent \
--set initContainers[0].name=velero-plugin-for-aws \
--set initContainers[0].image=velero/velero-plugin-for-aws:v1.0.0 \
--set initContainers[0].volumeMounts[0].mountPath=/target \
--set initContainers[0].volumeMounts[0].name=plugins \
vmware-tanzu/velero
 ``` 
 # install flux 
 forker ce repo
 https://github.com/soufianem370/flux-get-started/settings/keys
 
Add the Flux repository:
```
helm repo add fluxcd https://charts.fluxcd.io
```
Apply the Helm Release CRD:
```
kubectl apply -f https://raw.githubusercontent.com/fluxcd/helm-operator/master/deploy/crds.yaml
```
In this next step you install Flux using helm. Simply

Fork fluxcd/flux-get-started on GitHub and replace the fluxcd with your GitHub username in here

Create the flux namespace:

```
kubectl create namespace flux
```
Install Flux and the Helm Operator by specifying your fork URL:

Just make sure you replace YOURUSER with your GitHub username in the command below:

Using a public git server from bitbucket.com, github.com, gitlab.com, dev.azure.com, or vs-ssh.visualstudio.com:
```
helm upgrade -i flux fluxcd/flux \
--set git.url=git@github.com:soufianem370/flux-get-started \
--namespace flux
```
```
helm upgrade -i helm-operator fluxcd/helm-operator \
--set git.ssh.secretName=flux-git-deploy \
--namespace flux
```
Note: By default the helm-operator chart will install with support for both Helm v2 (which requires Tiller) and v3. You can target specific versions by setting the helm.versions value, e.g. --set helm.versions=v3.

Using a private git server:
When deploying from a private repo, the known_hosts of the git server needs to be configured into a kubernetes configmap so that StrictHostKeyChecking is respected. See the README of the chart for further installation instructions in this case.

Allow some time for all containers to get up and running. If you're impatient, run the following command and see the pod creation process.

```bash
watch kubectl -n flux get pods
```
You will notice that flux and flux-helm-operator will start turning up in the flux namespace.

## install fluxctl
https://snapcraft.io/install/fluxctl/centos


### Using Kind to deploy a Kubernetes Cluster

1- Download from golang.org the package file, which correspond to your OS
For linuxOS
```bash
wget https://dl.google.com/go/go1.14.2.linux-amd64.tar.gz
``` 
2- Extract that file to /usr/local 
```bash
tar xfzv go1.14.2.linux-amd64.tar.gz -C /usr/local
``` 
3- export Go package to your current path:
```bash
export PATH=$PATH:/usr/local/go/bin
```
4- access the kubernetes-sigs/kind repo, there is a single command you need tu run 
```bash
GO111MODULE="on" go get sigs.k8s.io/kind@v0.7.0
```
5- copy kind binary to your path directory or export its the present path
```bash
cp /go/bin/kind /usr/local/bin/kind 
# or
export PATH=$PATH:/home/user/go/bin
```
6- Command you should to know about kind
```bash
which kind
kind version
kind help kind help create 
kind create cluster 
kind create cluster --config config-file-path
kind delete cluster
```
7- the command "kind create cluster" will create just one master nodes, if you want to create customized config, you need just to create a yaml file, than choose the config you find appropriate, to get several models of theses configuration:
```bash
# three node (two workers) cluster config
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```
Réf: https://kind.sigs.k8s.io/docs/user/quick-start#creating-a-cluster
N.B: 
```bash
kind create cluster --config config-file.yaml
```
N.B: In case if you want to deploy a high availibility cluster with 03 master or more and workers, kind will ensure with theses nodes another node which will be deployed therefore ensuring the HA role (HA Proxy).

### Deploy an ELK slack on Kuberenetes Cluster
         
ELK:
     - ElasticSearch: Storage Engin
     - Logstach: Receive and store logs in ElasticSearch
     - kibana: virtualise the logs - dashboard - report 
1- ElasticSearch Pre-requisistes:
Java 8 - Port 9200 - Port 9300 (Node de communication) - config file: /etc/elasticsearch/elasticsearch.yaml - Log files: /var/log/elasticsearch 
2- Logstach Pre-requisistes: Port 5044 - config-file: /etc/logstach/conf.d/01-logstach-simple.conf - log-files: journal enabled 
3- Kibana Pre-requisistes: Port 5601 - Use Nginx as reverse proxy - config-file /etc/kibana/kibana.yml - log-files: journal enabled

## Setup Components
following steps in quickstart ELK website:
```bash
https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-quickstart.html
```
Using port/forward ALLOW to access theses component just from the localhost, so to access logstash and kibana trough a browser from an outside machine you must modify each svc for each component from ClusterIP type to NodePort or to LoadBalancer if you have already installed one metalLb for eg.

 
