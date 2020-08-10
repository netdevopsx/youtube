# Installation Process
## Loging into Linux server
```
ssh ubuntu@192.168.0.10
```

## Install fly
```
curl -LO https://github.com/concourse/concourse/releases/download/v6.3.0/fly-6.3.0-linux-amd64.tgz
tar zxvf fly-6.3.0-linux-amd64.tgz
chmod +x fly
sudo chown root:root fly
sudo mv fly /usr/local/bin/
```

## Login to concourse and create pipelines
Clone repository 
git clone https://github.com/netdevopsx/awx_install_ci.git
cd awx_install_ci
Login to Concourse
```
fly --target netdevopsx login --team-name main \
    --concourse-url https://ci.netdevopsx.com
```
Create pipelines
```
fly -t netdevopsx set-pipeline \
    --pipeline awx_deploy \
    --config pipelines/awx_deploy.yml

fly -t netdevopsx set-pipeline \
    --pipeline concourse_images \
    --config pipelines/concourse_images.yml
```
## Create secrets

### Private key for github.com (Manage version of images)
```
ssh-keygen -f concourse_ci 
kubectl -n concourse-main create secret generic github-version --from-file=private_key=concourse_ci
```
concourse_ci.pub should be uploaded to settings of repository awx_install_ci

### Kube-config
```
kubectl -n concourse-main create secret generic kube-config2 --from-file=value=/home/ubuntu/.kube/config
```

## Build the images (concourse-kubernetes, concourse-ansible)
Navigate to concourse_images and trigger patch, then both image's jobs: concourse-kubernetes and concourse-ansible

## Deploy AWX
Navigate to awx_deploy and trigger jobs: awx-green-install and awx-brue-install then awx-green-provision and awx-blue-provision

## Select which colour will be production one
Triger one colour blue or green