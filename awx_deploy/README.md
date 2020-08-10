# Installation Process
## Loging into Linux server

ssh ubuntu@192.168.0.10

## Install fly
curl -LO https://github.com/concourse/concourse/releases/download/v6.3.0/fly-6.3.0-linux-amd64.tgz
tar zxvf fly-6.3.0-linux-amd64.tgz
chmod +x fly
sudo chown root:root fly
sudo mv fly /usr/local/bin/

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