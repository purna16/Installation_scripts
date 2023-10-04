***Installing kubernetes complete procedure using kubeadmin tool***

- we need to visit official documentation of kubernetes for installation [Documentation]( https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

**first we need to setup ipv4 forwarding rules befpre installing on control plane and nodes**
```
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
- These commands help ensure that your system is correctly configured to support the networking and container runtime requirements of Kubernetes. They are typically required steps during the initial setup of a Kubernetes cluster to ensure that pods can communicate with each other and with the external world. The specific settings may vary based on the container runtime and networking solution you're using.

- We need to update apt repository and these packages are often necessary for setting up secure connections and fetching data from remote repositories when installing software or updates on a Linux system
```
sudo apt-get install -y apt-transport-https ca-certificates curl
```
**add the google cloud public signing key on both nodes**
- Adding the Google Cloud GPG key ensures that the Kubernetes packages you install are authentic and haven't been tampered with during transmission. Without this verification step, there would be a risk of installing malicious or corrupted software.
```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
- we will get an error because newer versions of ubuntu dont have a repository created(/etc/apt/keyrings) we need to create that repo and run command
  
**we need to add Kubernetes apt repository on both nodes**
- By adding this repository, you make Kubernetes packages available for installation and updates through your system's package manager (apt-get). This simplifies the process of managing and updating your Kubernetes installation, and it also ensures that you get official, signed packages from the Kubernetes project.
```
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
**install kubelet, kubeadm and kubectl, and pin their version:**
```
sudo apt-get update
sudo apt-get install -y kubelet=1.27.0-00 kubeadm=1.27.0-00 kubectl=1.27.0-00
sudo apt-mark hold kubelet kubeadm kubectl

```
- This command is used to prevent specific packages (in this case, kubelet, kubeadm, and kubectl) from being automatically upgraded when you run system package updates using apt-get
  
**we need to run kubeadm init now to initialize cluster**
```
kubeadm init --apiserver-advertise-address 192.31.21.12 --pod-network-cidr 10.244.0.0/16 --apiserver-cert-extra-sans controlplane

```
- --apiserver-advertise-address 192.31.21.12 The API server will tell all the other parts of the cluster to contact it at this address for instructions and management
- we know about pod network as they come under that specified network range
- --apiserver-cert-extra-sans controlplane Adding this flag can be useful in cases where you want to access the Kubernetes API server using this name (e.g., https://controlplane:6443), and you want the certificate to be valid for this name

***once done we have to set kubeconfig file**
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
***now take the given token by kubeadmin tool and run it on worker nodes**
```
kubeadm join 192.31.21.12:6443 --token n8atqp.ebpxafxu33un2ew3 \
        --discovery-token-ca-cert-hash sha256:f74bc6a4fc51da3decdf9e9b0c094a6547857beac9eb0e4149efcff328a0f36e 
```

***final step is to configure network***
- if we check terminal we can see there is a link to networks we just need to open it or search it on documentation
- we use flannel as default network
```
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```


```

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system

sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

mkdir -p /etc/apt/keyrings

curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update

sudo apt-get install -y kubelet=1.27.0-00 kubeadm=1.27.0-00 kubectl=1.27.0-00

sudo apt-mark hold kubelet kubeadm kubectl

kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```
