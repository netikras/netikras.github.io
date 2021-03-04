
# Kubernetes - quickly set up your own cluster

Want to quickly spin up your own Kubernetes cluster? Here's how.

- install docker
- install kubeadm, kubectl
- disable swap
- kubeadm init
- kubeadm join
- kubectl get nodes
- install network fabric, e.g. weave
- install nginx controller
- get ingess controller's Endpoint
- install helm

This is what it looks like


```

                                      [ CLIENT ]
                                          ↓
                                 [ Ingress Endpoint ] NodePort :80/tcp, :443/tcp
                                          ↓
                                [ Ingress Controller ] (nginx pod)
            ______________________________|______________________________
            ↓                             ↓                             ↓
     [ ingress-app1 ]  /app1/*     [ ingress-app2 ] /app2/*      [ ingress-app3 ] /app3/*
            ↓                             ↓                             ↓
     [ endpoint-app1 ] :8080/tcp   [ endpoint-app2 ] :8080/tcp   [ endpoint-app3 ] :8080/tcp
            ↓                             ↓                             ↓
       [ svc-app1 ]                  [ svc-app2 ]                  [ svc-app3 ]
    ________|________             ________|________             ________|________
    ↓       ↓       ↓             ↓       ↓       ↓             ↓       ↓       ↓
  [pod1]  [pod2]  [pod3]        [pod1]  [pod2]  [pod3]        [pod1]  [pod2]  [pod3]
    ↓       ↓       ↓             ↓       ↓       ↓             ↓       ↓       ↓
 [cont0] [cont0] [cont0]       [cont0] [cont0] [cont0]       [cont0]  [cont0] [cont0]
 
```



## Install Docker
### Ubuntu

```bash

sudo apt-get update
sudo apt-get install docker-ce
sudo systemctl status docker

```

## Install kubeadm and kubectl

### kubectl

[kubernetes.io link](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

##### generic linux
  
```bash

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo mv kubectl /usr/bin/kubectl
sudo chown root:root /usr/bin/kubectl
sudo chmod 0755 /usr/bin/kubectl
kubectl version --client

```

---

##### ubuntu
  
```bash

sudo apt-get update && sudo apt-get install -y apt-transport-https gnupg2 curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl
kubectl version --client

```

---

##### RH/CentOS
  
```bash

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
sudo yum install -y kubectl
kubectl version --client

```

### kubeadm
[kubernetes.io link](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

##### generic linux

Install CNI plugins (required for most pod network):

```bash

CNI_VERSION="v0.8.2"
sudo mkdir -p /opt/cni/bin
curl -L "https://github.com/containernetworking/plugins/releases/download/${CNI_VERSION}/cni-plugins-linux-amd64-${CNI_VERSION}.tgz" | sudo tar -C /opt/cni/bin -xz

```

Define the directory to download command files

> **Note:**  The  `DOWNLOAD_DIR`  variable must be set to a writable directory. If you are running Flatcar Container Linux, set  `DOWNLOAD_DIR=/opt/bin`.

```bash

DOWNLOAD_DIR=/usr/local/bin
sudo mkdir -p $DOWNLOAD_DIR

```

Install crictl (required for kubeadm / Kubelet Container Runtime Interface (CRI))

```bash

CRICTL_VERSION="v1.17.0"
curl -L "https://github.com/kubernetes-sigs/cri-tools/releases/download/${CRICTL_VERSION}/crictl-${CRICTL_VERSION}-linux-amd64.tar.gz" | sudo tar -C $DOWNLOAD_DIR -xz

```

Install  `kubeadm`,  `kubelet`,  `kubectl`  and add a  `kubelet`  systemd service:

```bash

RELEASE="$(curl -sSL https://dl.k8s.io/release/stable.txt)"
cd $DOWNLOAD_DIR
sudo curl -L --remote-name-all https://storage.googleapis.com/kubernetes-release/release/${RELEASE}/bin/linux/amd64/{kubeadm,kubelet,kubectl}
sudo chmod +x {kubeadm,kubelet,kubectl}

RELEASE_VERSION="v0.4.0"
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/kubepkg/templates/latest/deb/kubelet/lib/systemd/system/kubelet.service" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /etc/systemd/system/kubelet.service
sudo mkdir -p /etc/systemd/system/kubelet.service.d
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/kubepkg/templates/latest/deb/kubeadm/10-kubeadm.conf" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

```

Enable and start `kubelet`:

```bash

sudo systemctl enable --now kubelet

```

---

##### Ubuntu

```bash

sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

```

The kubelet is now restarting every few seconds, as it waits in a crashloop for kubeadm to tell it what to do.

---

##### RH/CentOS

```bash

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet

```

**Notes:**

-   Setting SELinux in permissive mode by running  `setenforce 0`  and  `sed ...`  effectively disables it. This is required to allow containers to access the host filesystem, which is needed by pod networks for example. You have to do this until SELinux support is improved in the kubelet.
    
-   You can leave SELinux enabled if you know how to configure it but it may require settings that are not supported by kubeadm.
    

The kubelet is now restarting every few seconds, as it waits in a crashloop for kubeadm to tell it what to do.

---

## Disable SWAP

```bash

sudo swapoff -a
sudo sed -i.bak -r 's/(.+ swap .+)/#\1/' /etc/fstab

```

## Initialize control plane (master node)

```bash

sudo kubeadm init

```

work around the problems (if any logged)

note down the `kubeadm join <...>` command in the output

```bash

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```

P.S. Should you need to undo any effects of kubeadm, run

```bash

sudo kubeadm reset

```

## Join cluster nodes

```bash

ssh remote_server -l username
## install kubeadm
## disable swap
## run the complete `kubeadm join <...>` command acquired from the `kubeadm init` output in the control plane

```

If any errors logged - resolve them as you go

## Verify nodes
In control plane

```bash

kubectl get nodes

```

all nodes should be **Ready**.

## Install network fabric
There are multiple network implementations: flannel, weave, calico, etc.

Flannel or weave ar both perfectly fine for a local setup

### Flannel
[link](https://github.com/flannel-io/flannel)

For Kubernetes v1.17+  

```bash

kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

```

---

### Weave
[link](https://www.weave.works/blog/weave-net-kubernetes-integration/)

Kubernetes versions 1.6 and above:

```bash

kubectl apply -f "[https://cloud.weave.works/k8s/net?k8s-version=$(kubectl](https://www.google.com/url?q=https://cloud.weave.works/k8s/net?k8s-version%3D$(kubectl&sa=D&ust=1524760785241000&usg=AFQjCNF4lxXiYNW0iq1-mOap4Wr7Ei9-YA) version | base64 | tr -d '\n')"

```

Kubernetes versions up to 1.5:

```bash

kubectl apply -f https://git.io/weave-kube

```

---

### Calico
[link](https://docs.projectcalico.org/getting-started/kubernetes/quickstart)
1.  Install the Tigera Calico operator and custom resource definitions.

	```bash
	
	kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
	
	```

2.  Install Calico by creating the necessary custom resource. For more information on configuration options available in this manifest, see  [the installation reference](https://docs.projectcalico.org/reference/installation/api).
	
	```bash
	
	kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
	
	```

	> **Note**: Before creating this manifest, read its contents and make sure its settings are correct for your environment. For example, you may need to change the default IP pool CIDR to match your pod network CIDR.

3.  Confirm that all of the pods are running with the following command.
	
	```bash
	
	watch kubectl get pods -n calico-system
	
	```

	Wait until each pod has the  `STATUS`  of  `Running`.

	> **Note**: The Tigera operator installs resources in the  `calico-system`  namespace. Other install methods may use the  `kube-system`  namespace instead.
	    
4.  Remove the taints on the master so that you can schedule pods on it.
	
	```bash
	
	kubectl taint nodes --all node-role.kubernetes.io/master-
	
	```

	It should return the following.
	
	```bash
	
	node/<your-hostname> untainted
	
	```
	
5.  Confirm that you now have a node in your cluster with the following command.
	
	```bash
	
	kubectl get nodes -o wide
	
	```
	
	It should return something like the following.
	
	```bash
	
	NAME              STATUS   ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION    CONTAINER-RUNTIME
	<your-hostname>   Ready    master   52m   v1.12.2   10.128.0.28   <none>        Ubuntu 18.04.1 LTS   4.15.0-1023-gcp   docker://18.6.1
	
	```

---
    
    
## Install Ingress Controller (nginx)
[link](https://kubernetes.github.io/ingress-nginx/deploy/)
#### [Bare-metal](https://kubernetes.github.io/ingress-nginx/deploy/#bare-metal)
Using  [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport):

```bash

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.44.0/deploy/static/provider/baremetal/deploy.yaml

```

Using HELM

```bash

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx

```

## Get ingess controller's Endpoint

```bash

kubectl -n ingress-nginx get svc | grep NodePort

```

Use this IP to query your ingresses

## Install helm
[link](https://helm.sh/docs/intro/install/)

##### generic linux

```bash

curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

```

---

##### ubuntu

```bash

curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm

```

---

## Done
Now deploy your pods, services and ingresses and access them through the NodePort endpoint's IP:port.



> Written with [StackEdit](https://stackedit.io/).
