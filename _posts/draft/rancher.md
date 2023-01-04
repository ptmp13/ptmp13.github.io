---
layout:     post
title:      "RKE2+Rancher+Calico+Traefik installation on CentOS 8"
subtitle:   " \"Ποσειδῶν...\""
date:       2022-12-20 16:49:00
author:     "ptmp13"
catalog: true
tags:
    - rke2
    - rancher
---

### Install Server

```bash
yum install nc mc vim net-tools -y
```

#### Add repos (now lates __1.26__) and install rke2-server

```bash
cat << EOF > /etc/yum.repos.d/rancher-rke2-1-18-latest.repo
[rancher-rke2-common-latest]
name=Rancher RKE2 Common Latest
baseurl=https://rpm.rancher.io/rke2/latest/common/centos/8/noarch
enabled=1
gpgcheck=1
gpgkey=https://rpm.rancher.io/public.key

[rancher-rke2-1-24-latest]
name=Rancher RKE2 1.24 Latest
baseurl=https://rpm.rancher.io/rke2/latest/1.24/centos/8/x86_64
enabled=1
gpgcheck=1
gpgkey=https://rpm.rancher.io/public.key
EOF

yum -y install rke2-server
```

#### Edit _/etc/rancher/rke2/config.yaml_

```bash
write-kubeconfig-mode: "0644"
write-kubeconfig: "/root/.kube/config"
cluster-cidr: 10.42.0.0/16
service-cidr: 10.43.0.0/16
cluster-dns: 10.43.0.10
cluster-domain: weblogic.local
data-dir: /media/data/rancher/rke2
image-credential-provider-bin-dir: /media/data/rancher/credentialprovider/bin
image-credential-provider-config: /media/data/rancher/credentialprovider/config.yaml
cni: "calico"
token: weblogic-adf-token
disable:
  - rke2-canal
  - rke2-kube-proxy
  - rke2-ingress-nginx
node-name: poseidons01.ru-central1.internal
node-label:
  - "status=master"
tls-san:
  - poseidon01
  - 10.129.0.22
```

#### Run 

```bash
setenforce 0 
sed -i 's/' /etc/sysconfig/selinux
systemctl enable rke2-server
systemctl start rke2-server
```

#### Some commands for comfort work

Add aliases
```bash
cd /usr/bin/
ln -s /media/data/rancher/rke2/bin/kubectl kubectl
ln -s /media/data/rancher/rke2/bin/crictl crictl
```

Add CONFIGS too bash
```bash
yum install bash-completion -y
echo 'source <(kubectl completion bash)' >>~/.bashrc

cat << EOF >> ~/.bash_profile
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
export CRI_CONFIG_FILE=/media/data/rancher/rke2/agent/etc/crictl.yaml
EOF

# Check
kubectl get po
crictl ps
```

Get nodes

```bash
kubectl get nodes
```

Using crictl

```bash
crictl ps
```

### Install Agent

#### Add repos (now lates __1.26__)

```bash
cat << EOF > /etc/yum.repos.d/rancher-rke2-1-18-latest.repo
[rancher-rke2-common-latest]
name=Rancher RKE2 Common Latest
baseurl=https://rpm.rancher.io/rke2/latest/common/centos/8/noarch
enabled=1
gpgcheck=1
gpgkey=https://rpm.rancher.io/public.key

[rancher-rke2-1-24-latest]
name=Rancher RKE2 1.24 Latest
baseurl=https://rpm.rancher.io/rke2/latest/1.24/centos/8/x86_64
enabled=1
gpgcheck=1
gpgkey=https://rpm.rancher.io/public.key
EOF

yum -y install rke2-agent
```

#### Edit _/etc/rancher/rke2/config.yaml_
```bash
server: https://poseidon01.ru-central1.internal:9345
token: weblogic-adf-token
data-dir: /media/data/rancher/rke2
node-label:
  - "status=agent"
tls-san:
  - triton01.ru-central1.internal
  - triton01
  - 10.129.0.24

systemctl start rke2-agent.service
```

### Install rancher

Install helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

Install traefic
```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
kubectl create ns traefik
helm install --namespace=traefik traefik traefik/traefik
```


Install rancher with helm

```bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo list
kubectl create namespace cattle-system

kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.7.1/cert-manager.crds.yaml

helm repo add jetstack https://charts.jetstack.io

helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.7.1

helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=51.250.97.147.sslip.io \
  --set replicas=1 \
  --set bootstrapPassword=pfyxth \
  --version=2.7.0
```

### Install VNC (not)

```bash
dnf groupinstall "Server with GUI"
systemctl isolate graphical
dnf install tigervnc-server
useradd vncpeter
passwd vncpeter
su - vncpeter
vncpasswd
vncserver :1
```



### Remove ALL

rke2 server --cluster-reset
rm -rf /var/lib/rancher
rm -rf /run/k3s

# Reset Password

kubectl --kubeconfig $KUBECONFIG -n cattle-system exec $(kubectl --kubeconfig $KUBECONFIG -n cattle-system get pods -l app=rancher | grep '1/1' | head -1 | awk '{ print $1 }') -- reset-password


PS:
openssl get pass: 0u6FJwogGu3JPbjxYrSg


        securityContext:
          allowPrivilegeEscalation: true
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - ALL
