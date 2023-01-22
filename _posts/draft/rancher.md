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
node-name: poseidon01
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
  --set hostname=134.122.64.221.sslip.io \
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


### Mod nginx igress DaemonSet for specific host

Label node:
```bash
kubectl label node poseidon-nginx ingress=nginx
```

Attach DaemonSet to node
```bash
kubectl edit ds -n kube-system rke2-ingress-nginx-controller
#add to nodeSelector:
# ingress=nginx
```

In deployment we have ports

```yaml
  - containerPort: 15021
    protocol: TCP
  - containerPort: 8080
    protocol: TCP
  - containerPort: 8443
    protocol: TCP
  - containerPort: 31400
    protocol: TCP
  - containerPort: 15443
    protocol: TCP
  - containerPort: 15090
    name: http-envoy-prom
    protocol: TCP
```

In service
```yaml
  - name: status-port
    nodePort: 32714
    port: 15021
    protocol: TCP
    targetPort: 15021
  - name: http2
    nodePort: 31380
    port: 80
    protocol: TCP
    targetPort: 8080
  - name: https
    nodePort: 31390
    port: 443
    protocol: TCP
    targetPort: 8443
  - name: tcp
    nodePort: 31400
    port: 31400
    protocol: TCP
    targetPort: 31400
  - name: tls
    nodePort: 30368
    port: 15443
    protocol: TCP
    targetPort: 15443
```

In generate file:
```yaml
            - containerPort: 15021
              protocol: TCP
            - containerPort: 8080
              protocol: TCP
            - containerPort: 8443
              protocol: TCP
            - containerPort: 15090
              protocol: TCP
              name: http-envoy-prom
```


```bash
istioctl manifest generate > 1.yaml
cd manifests/charts/gateways/istio-ingress
helm template istio . --output-dir istio-manifests
```


Modify:
  namespace: istio-system

Ports to:
  - containerPort: 15021
    protocol: TCP
  - containerPort: 8080
    hostPort: 80
    protocol: TCP
  - containerPort: 8443
    hostPort: 443
    protocol: TCP
  - containerPort: 31400
    protocol: TCP
  - containerPort: 15443
    protocol: TCP
  - containerPort: 15090
    name: http-envoy-prom
    protocol: TCP
+
nodeAffinity

Result Istio DaemonSet will be:
```yaml
---
# Source: istio-ingress/templates/deployment.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: istio-ingressgateway
  namespace: istio-system
  labels:
    app: istio-ingressgateway
    istio: ingressgateway
    release: istio
    istio.io/rev: default
    install.operator.istio.io/owning-resource: unknown
    operator.istio.io/component: "IngressGateways"
spec:
  selector:
    matchLabels:
      app: istio-ingressgateway
      istio: ingressgateway
  template:
    metadata:
      labels:
        app: istio-ingressgateway
        istio: ingressgateway
        service.istio.io/canonical-name: istio-ingressgateway
        service.istio.io/canonical-revision: latest
        istio.io/rev: default
        install.operator.istio.io/owning-resource: unknown
        operator.istio.io/component: "IngressGateways"
        sidecar.istio.io/inject: "false"
      annotations:
        prometheus.io/port: "15020"
        prometheus.io/scrape: "true"
        prometheus.io/path: "/stats/prometheus"
        sidecar.istio.io/inject: "false"
    spec:
      securityContext:
        runAsUser: 1337
        runAsGroup: 1337
        runAsNonRoot: true
        fsGroup: 1337
      serviceAccountName: istio-ingressgateway-service-account
      containers:
        - name: istio-proxy
          image: "docker.io/istio/proxyv2:1.16.1"
          ports:
            - containerPort: 15021
              protocol: TCP
            - containerPort: 8080
              hostPort: 80
              protocol: TCP
            - containerPort: 8443
              hostPort: 443
              protocol: TCP
            - containerPort: 31400
              protocol: TCP
            - containerPort: 15443
              protocol: TCP
            - containerPort: 15090
              name: http-envoy-prom
              protocol: TCP
          args:
          - proxy
          - router
          - --domain
          - $(POD_NAMESPACE).svc.cluster.local
          - --proxyLogLevel=warning
          - --proxyComponentLogLevel=misc:error
          - --log_output_level=default:info
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
              - ALL
            privileged: false
            readOnlyRootFilesystem: true
          readinessProbe:
            failureThreshold: 30
            httpGet:
              path: /healthz/ready
              port: 15021
              scheme: HTTP
            initialDelaySeconds: 1
            periodSeconds: 2
            successThreshold: 1
            timeoutSeconds: 1
          resources:
            limits:
              cpu: 2000m
              memory: 1024Mi
            requests:
              cpu: 100m
              memory: 128Mi
          env:
          - name: JWT_POLICY
            value: third-party-jwt
          - name: PILOT_CERT_PROVIDER
            value: istiod
          - name: CA_ADDR
            value: istiod.istio-system.svc:15012
          - name: NODE_NAME
            valueFrom:
              fieldRef:
                apiVersion: v1
                fieldPath: spec.nodeName
          - name: POD_NAME
            valueFrom:
              fieldRef:
                apiVersion: v1
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                apiVersion: v1
                fieldPath: metadata.namespace
          - name: INSTANCE_IP
            valueFrom:
              fieldRef:
                apiVersion: v1
                fieldPath: status.podIP
          - name: HOST_IP
            valueFrom:
              fieldRef:
                apiVersion: v1
                fieldPath: status.hostIP
          - name: SERVICE_ACCOUNT
            valueFrom:
              fieldRef:
                fieldPath: spec.serviceAccountName
          - name: ISTIO_META_WORKLOAD_NAME
            value: istio-ingressgateway
          - name: ISTIO_META_OWNER
            value: kubernetes://apis/apps/v1/namespaces/default/deployments/istio-ingressgateway
          - name: ISTIO_META_MESH_ID
            value: "cluster.local"
          - name: TRUST_DOMAIN
            value: "cluster.local"
          - name: ISTIO_META_UNPRIVILEGED_POD
            value: "true"
          - name: ISTIO_META_CLUSTER_ID
            value: "Kubernetes"
          volumeMounts:
          - name: workload-socket
            mountPath: /var/run/secrets/workload-spiffe-uds
          - name: credential-socket
            mountPath: /var/run/secrets/credential-uds
          - name: workload-certs
            mountPath: /var/run/secrets/workload-spiffe-credentials
          - name: istio-envoy
            mountPath: /etc/istio/proxy
          - name: config-volume
            mountPath: /etc/istio/config
          - mountPath: /var/run/secrets/istio
            name: istiod-ca-cert
          - name: istio-token
            mountPath: /var/run/secrets/tokens
            readOnly: true
          - mountPath: /var/lib/istio/data
            name: istio-data
          - name: podinfo
            mountPath: /etc/istio/pod
          - name: ingressgateway-certs
            mountPath: "/etc/istio/ingressgateway-certs"
            readOnly: true
          - name: ingressgateway-ca-certs
            mountPath: "/etc/istio/ingressgateway-ca-certs"
            readOnly: true
      volumes:
      - emptyDir: {}
        name: workload-socket
      - emptyDir: {}
        name: credential-socket
      - emptyDir: {}
        name: workload-certs
      - name: istiod-ca-cert
        configMap:
          name: istio-ca-root-cert
      - name: podinfo
        downwardAPI:
          items:
            - path: "labels"
              fieldRef:
                fieldPath: metadata.labels
            - path: "annotations"
              fieldRef:
                fieldPath: metadata.annotations
      - name: istio-envoy
        emptyDir: {}
      - name: istio-data
        emptyDir: {}
      - name: istio-token
        projected:
          sources:
          - serviceAccountToken:
              path: istio-token
              expirationSeconds: 43200
              audience: istio-ca
      - name: config-volume
        configMap:
          name: istio
          optional: true
      - name: ingressgateway-certs
        secret:
          secretName: "istio-ingressgateway-certs"
          optional: true
      - name: ingressgateway-ca-certs
        secret:
          secretName: "istio-ingressgateway-ca-certs"
          optional: true
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution: 
                nodeSelectorTerms:
                - matchExpressions:
                  - key: ingress
                    operator: In
                    values:
                    - istio
          preferredDuringSchedulingIgnoredDuringExecution:
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



=======
10.80.130.31 rancher-server-1
10.80.130.32 rancher-server-2
10.80.130.33 rancher-agent-1
10.80.130.34 rancher-vip-1

 

https://10.80.130.31:9090/
https://10.80.130.32:9090/
https://10.80.130.33:9090/

 

root/стандартный пароль как у oracle