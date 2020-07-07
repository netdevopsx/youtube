# Installation Process
## Clean server install
- Insert Ubuntu 18.04 CD
- Conduct traditional Ubuntu installation
- sudo vi /etc/netplan/00-installer-config.yaml
# Set static IP address on interface
```
network:
  ethernets:
    ens160:
      dhcp4: false
      dhcp6: false
      addresses:
        - 192.168.0.10/24
        - 192.168.0.11/24
        - 2a02:8071:31b6:8800::11/128
      gateway4: 192.168.0.1
      gateway6: 2a02:8071:31b6:8800::4998
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
          - 2001:4860:4860::8888
          - 2001:4860:4860::8844
  version: 2
```
- Reboot server
## Prepare terminal (SSH keys)
- Login to server
- ssh ubuntu@192.168.0.10
- ssh-keygen
- ssh-copy-id ubuntu@192.168.0.10
- sudo cp /home/ubuntu/.ssh/authorized_keys /root/.ssh/authorized_keys
# Install Kebernetes
## Download kubesprey and configure it
- git clone -b v2.13.2 https://github.com/kubernetes-sigs/kubespray.git
- cd kubespray
- sudo apt-get install virtualenv python3-pip -y
- virtualenv -p python3 ~/venv/ansible
- source ~/venv/ansible/bin/activate
- pip3 install -r requirements.txt
- cp -rfp inventory/sample inventory/mycluster
- declare -a IPS=(192.168.0.10)
- CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
- Rename servername in the inventory
- sed -i 's/node1/kube-master/g' inventory/mycluster/hosts.yaml
## Upgrade based software  
- sudo apt-get upgrade -y
# Get Kubesprey and prepare inventory

# Do some things under the root user

## Prepare additional disk for path provision
# add new disk to VM
ssh root@192.168.0.10
mkfs.ext4 /dev/sdb
DISK_UUID=$(blkid -s UUID -o value /dev/sdb)
mkdir -p /opt/local-path-provisioner/
mount /dev/sdb /opt/local-path-provisioner/
echo "UUID=$DISK_UUID /opt/local-path-provisioner/ ext4 defaults 0 0" >> /etc/fstab

## Install Helm 3

curl https://helm.baltorepo.com/organization/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm

# Install Kubernetes + local_path_provisioner_enabled

ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root cluster.yml \
-e "ansible_ssh_user=root \
local_path_provisioner_enabled=true \
dashboard_enabled=false"

# get kubernetes config file
mkdir  ~/.kube
scp root@192.168.0.10:/root/.kube/config ~/.kube/config

# we have only one node,  we don't need to more one pod for core-dns
# we will remove dns-autoscaler
kubectl delete deployment dns-autoscaler --namespace=kube-system
# scale current count of replicas to 1
kubectl scale deployments.apps -n kube-system coredns --replicas=1


# Install ingress for baremetal (NodePort)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/deploy.yaml


kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v0.15.1 \
  --set installCRDs=true

create clusterissuer

# Create secret for ClusterIssuer Secret must be in the namespace cert-manager
kubectl -n cert-manager create secret generic prod-route53-credentials-secret --from-literal=secret-access-key=MmtqhmKtcNIQJSe1CdSLvbkpKbQropF/Gcu6wuGU

kubectl create ns concourse
# Install Concourse
helm repo add concourse https://concourse-charts.storage.googleapis.com/
helm repo update
cd ..
git clone https://github.com/netdevopsx/youtube.git
cd youtube/kubernetes_ubuntu_18
vi concourse-value.yml
helm -n concourse install concourse -f concourse-value.yml concourse/concourse


Add Issuer
git clone https://github.com/netdevopsx/youtube.git
cd kubernetes_ubuntu_18
vi clusterissuer.yml (Private data)
kubectl apply -f clusterissuer.yml
vi cert-manager-secret.yml
kubectl apply -f cert-manager-secret.yml

ssh root@192.168.0.10
apt-get install haproxy
vi /etc/haproxy/haproxy.cfg

frontend front_http
        bind 192.168.0.11:80,[2a02:8071:31b6:8800::11]:80
        mode tcp
        default_backend back_http

backend back_http
        mode tcp
        balance roundrobin
        server http1 192.168.0.10:32292 check

frontend front_https
        bind 192.168.0.11:443,[2a02:8071:31b6:8800::11]:443
        mode tcp
        default_backend back_https

backend back_https
        mode tcp
        balance roundrobin
        server https1 192.168.0.10:31783 check

service haproxy restart
