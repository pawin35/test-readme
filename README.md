# Give Me Money
**Software Architecture Term Project:** Create Microservice

**Software-Defined System Term Project:** Build the Kubernetes cluster using Raspberry PI with this project

## Project structure
- database
- frontend
- service: Each service has the user for accessing only its database
  - user
    - Get all user information
    - Using `gmm_user` database
  - debt 
    - Get, Create the debt list
    - Using `gmm_debt` database
  - transaction 
    - Get, Create the transaction for paying debt
    - Using `gmm_transaction` database

## Build
For testing in local
- Build image and Create the container 
```sh
docker-compose up -d --build
```
- Open the browser with docker ip. For example, `172.17.0.1` or `localhost`
- Stop and Remove the container
```sh
docker-compose down
```

## Topology
Kubenetes cluster topology

![Alt text](image/topology.png?raw=true "Kubernetes cluster topology image")

| Device                                   |        Host name        |             IP Address             |
| ---------------------------------------- | ---------------------------------------- | :--------------------------------: |
| Master 1 [VM]                            |      kubernetes-master-0      |           192.168.0.254            |
| Master 2 [VM]                            |      kubernetes-master-1      |           192.168.0.253            |
| Load Balancer for Master [VM] - HA Proxy |       load-balancer-1       |           192.168.0.200            |
| Load Balancer for Client [VM] - nginx    |     client-load-balancer-1    |           192.168.0.100            |
| Worker Node [Raspberry PI]               | any describtive name |192.168.0.136, 192.168.0.142 - 144 |

## Set up Kubernetes cluster

### Set up Worker Node [Raspberry PI]

Set up process can be followed in [this medium article](https://medium.com/nycdev/k8s-on-pi-9cc14843d43). The difference is that our Raspberry Pi use Raspbian Buster, so we provide additional steps to allow each worker node to run correctly.

For Raspbian 10 (Debian 10)

- Swap need to be disbled for kubeadm to run correctly.

```sh
sudo swapoff -a
sudo systemctl disable dphys-swapfile.service
```

- Since latest kubeadm is not compatible with nftables, we have to change it to use legacy iptables. [See more detail](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#ensure-iptables-tooling-does-not-use-the-nftables-backend)

```sh
sudo apt install arptables

sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
sudo update-alternatives --set arptables /usr/sbin/arptables-legacy
sudo update-alternatives --set ebtables /usr/sbin/ebtables-legacy
```

### Preparing Master Node

We create multi-master using 2 VMs with stacked etcd-control plane strategy. The chosen linux distribution for master nodes is Debian 10 (Buster) as of Raspbian Buster was used in our Raspberry Pi nodes.

#### Install Debian for all master node

1. Download Debian 10 iso from [here](https://www.debian.org/CD/).
2. Install Debian on VirtualBox with 2GB of RAM or higher.
3. While installing, do not create swap to allow kubeadm to work correctly.
4. Configure VM to use bridged adapter as shown in the picture below.

![Alt text](image/setup-master/bridge-adapter.png?raw=true "Set network adapter type to bridged adapter")

> Important:
> Ensure that each master VM has different MAC address, product_uuid and hostname
> (Product uuid can be checked with `sudo cat /sys/class/dmi/id/product_uuid`)

#### Install and setup Docker

Docker installation process is based on [official docker-ce installation guide](https://docs.docker.com/install/linux/docker-ce/debian/).

1. Update the apt index with `sudo apt update`.
2. Enable https support for apt.
```sh
sudo apt-get install apt-transport-https ca-certificates curl gnupg2 software-properties-common
```
3. Add docker gpg key, add repository and install docker.
```sh
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/debian \
   $(lsb_release -cs) \
   stable"
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```
4. Add user to group docker with `sudo usermod -aG docker your-user`

#### Install and setup kubeadm and kubectl

Kubeadm installation process is based on [official kubeadm installation guide](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/).

1. Install iptables subsystem
```sh
sudo apt install iptables ip6tables arptables ebtables
```
2. Switch to legacy iptables subsystem as nftables is not compatible with kubeadm.
```sh
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
sudo update-alternatives --set arptables /usr/sbin/arptables-legacy
sudo update-alternatives --set ebtables /usr/sbin/ebtables-legacy
```
3. Install kubectl, kubelet and kubeadm
```sh
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Set up Load Balancer for master - HAProxy

#### Install Debian and basic setup

1. Set up the Debian using [the same process as in master nodes](#install-debian-for-all-master-node)
2. configuring IP and host name of the load balancer server according to the [topology of this project)(#topology)
3. Set up HAProxy

``` sh
sudo apt-get update
sudo apt-get install haproxy
```

#### Setting up HAProxy

We have to set up HAProxy as a load balancer so that it can distribute the controlling packet to each of the two master nodes in a round robin fashion.

1. Edit the HAProxy configuration file
``` sh
sudo vim /etc/haproxy/haproxy.cfg
```

2. Add the following information to the end of the configuration file. you should pay special attension to the host name and IP address of each master node.


```
frontend kubernetes
	bind 192.168.0.200:6443 #The IP address and port used for connecting to the load balancer
	option tcplog
	mode tcp
	default_backend kubernetes-master-nodes

backend kubernetes-master-nodes
	mode tcp
	balance roundrobin #Use round robin when selecting the master nodes
	option tcp-check
	server kubernetes-master-0 192.168.0.254:6443 check fall 3 rise 2 #The host name and IP address of first master node
	server kubernetes-master-1 192.168.0.253:6443 check fall 3 rise 2 #The host name and IP address of second master node
```

3. Restart the HAProxy

``` sh
sudo systemctl restart haproxy.service
```


### Initialize cluster

After load balancer for master nodes was set, the cluster with stacked etcd-control plane can be initialized. The process is based on 2 sources which is [this medium article](https://medium.com/nycdev/k8s-on-pi-9cc14843d43) and [official kubeadm HA cluster guide](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/).

> From now, we assume that master nodes' load balancer is at 192.168.0.200:6443

#### Initialize first master node

1. Init kubernetes cluster
```sh
sudo kubeadm init --control-plane-endpoint "192.168.0.200:6443" --upload-certs --token-ttl=0
```
2. Install weavenet
```sh
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
3. Setup kubectl
```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

> If you want to use kubectl on your other machine, use the same config in step 3 to set up kubectl on it.

#### Initialize second master node

Run the following command
```sh
sudo kubeadm join 192.168.0.200:6443 --token <your-token> --discovery-token-ca-cert-hash sha256:<your-discovery-token-ca-cert-hash> --control-plane --certificate-key <your-certificate-key>
```

> Your master nodes should be all set! Try getting nodes status with `kubectl get nodes`

#### Initialize each worker node

Run the following command
```sh
sudo kubeadm join 192.168.0.200:6443 --token <your-token> --discovery-token-ca-cert-hash sha256:<your-discovery-token-ca-cert-hash>
```

> From now, Your cluster should be now ready to deploy applications!

### Set up Load Balancer for client - nginx
1. Choose the VM in the notebook. For the VM, you can use whatever you would like such as Ubuntu or Debian
2. After selecting the VM in the VirtualBox, Go to `Settings > Network`. At the `Attached to:` in Adapter 1, Select `Bridged Adapter`
3. Install nginx
```sh
sudo apt-get update
sudo apt-get install nginx
```
4. In this project root directory, copy `nginx.conf` to `/etc/nginx/nginx.conf`
5. Remove the default symbolic link in site-enabled folder
```sh
sudo rm /etc/nginx/sites-enabled/default
```
6. Restart nginx
```sh
sudo systemctl restart nginx
```

## Deploy the application to Kubernetes cluster

1. IP address of client's load balancer in environment variables inside each deployment file in `k8s/deployment` have been set to use `192.168.0.100`.
2. Service files in `k8s/service` have been set `type` field to `NodePort` 
- `gmm-frontend.yaml` set `nodePort:32000`
- `gmm-service-user.yaml` set `nodePort:31999`
- `gmm-service-debt.yaml` set `nodePort:31998`
- `gmm-service-transaction.yaml` set `nodePort:31997`
- `gmm-database.yaml` set `nodePort:31996` 
3. Run `deploy.sh` in `k8s` folder to deploy the application
```sh
./deploy.sh
```



