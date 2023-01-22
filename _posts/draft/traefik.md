```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: traefik-role

rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
      - networking.k8s.io
    resources:
      - ingresses
      - ingressclasses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
      - networking.k8s.io
    resources:
      - ingresses/status
    verbs:
      - update
```

helm repo add traefik https://helm.traefik.io/traefik

helm repo add traefik https://traefik.github.io/charts
helm repo update



### Install 
kubectl create ns ingress-apisix
helm install apisix apisix/apisix \
  --set gateway.type=NodePort \
  --set ingress-controller.enabled=true \
  --namespace ingress-apisix \
  --set ingress-controller.config.apisix.serviceNamespace=ingress-apisix \
  --kubeconfig /etc/rancher/rke2/rke2.yaml

helm uninstall apisix apisix/apisix --kubeconfig /etc/rancher/rke2/rke2.yaml

NAME: apisix
LAST DEPLOYED: Wed Jan  4 14:04:35 2023
NAMESPACE: ingress-apisix
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. Get the application URL by running these commands:
  export NODE_PORT=$(kubectl get --namespace ingress-apisix -o jsonpath="{.spec.ports[0].nodePort}" services apisix-gateway)
  export NODE_IP=$(kubectl get nodes --namespace ingress-apisix -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT

[root@poseidon01 helm]# kubectl get po -n ingress-apisix 
NAME                                        READY   STATUS     RESTARTS   AGE
apisix-697646cb48-2dbwb                     0/1     Init:0/1   0          49s
apisix-etcd-0                               0/1     Pending    0          49s
apisix-etcd-1                               0/1     Pending    0          49s
apisix-etcd-2                               0/1     Pending    0          49s
apisix-ingress-controller-db4957957-rwnfc   0/1     Init:0/1   0          49s


kubectl create -f pv-apisix.yaml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: task-pv-volume
  labels:
    type: local
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/media/data/apisix"



helm install apisix-dashboard apisix/apisix-dashboard --namespace=ingress-apisix

1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace ingress-apisix -l "app.kubernetes.io/name=apisix-dashboard,app.kubernetes.io/instance=apisix-dashboard" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace ingress-apisix $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace ingress-apisix port-forward $POD_NAME 8080:$CONTAINER_PORT


## Update password
https://github.com/apache/apisix-dashboard/blob/master/api/conf/conf.yaml#L70-L75


### Errors 

```bash
[root@poseidon01 helm]# kubectl logs apisix-etcd-0 -n ingress-apisix 
etcd 15:16:49.72 
etcd 15:16:49.73 Welcome to the Bitnami etcd container
etcd 15:16:49.73 Subscribe to project updates by watching https://github.com/bitnami/containers
etcd 15:16:49.73 Submit issues and feature requests at https://github.com/bitnami/containers/issues
etcd 15:16:49.73 
etcd 15:16:49.73 INFO  ==> ** Starting etcd setup **
etcd 15:16:49.75 INFO  ==> Validating settings in ETCD_* env vars..
etcd 15:16:49.75 WARN  ==> You set the environment variable ALLOW_NONE_AUTHENTICATION=yes. For safety reasons, do not use this flag in a production environment.
etcd 15:16:49.76 INFO  ==> Initializing etcd
etcd 15:16:49.76 INFO  ==> Generating etcd config file using env variables
etcd 15:16:49.78 INFO  ==> There is no data from previous deployments
etcd 15:16:49.78 INFO  ==> Bootstrapping a new cluster
etcd 15:17:49.97 ERROR ==> Headless service domain does not have an IP per initial member in the cluster
```

Check folder permissions:

```bash
chmod 777 /media/data/apisix
```


### Uninstall
helm uninstall apisix -n ingress-apisix
kubectl delete pv task-pv-volume
kubectl patch pv task-pv-volume -p '{"metadata":{"finalizers":null}}'


ClientURLs:[http://apisix-etcd-0.apisix-etcd-headless.ingress-apisix.svc.cluster.local:2379 http://apisix-etcd.ingress-apisix.svc.cluster.local:2379]}","request-path":"/0/members/e2571e968b89c849/attributes","publish-timeout":"7s","error":"etcdserver: request timed out"}
