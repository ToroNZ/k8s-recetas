# k8s-recetas

## Dashboard ##

Deploy the dashboard in the following way:

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```
> **https://github.com/kubernetes/dashboard**

## Metrics ##

Heapster:

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/rbac/heapster-rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/standalone/heapster-controller.yaml
```

## Service Discovery ##

Istio (Envoy c++ proxy+Istio mesh):

Download Istio (0.4.0 in this case):
https://github.com/istio/istio/releases
```
wget -qO- https://github.com/istio/istio/releases/download/0.4.0/istio-0.4.0-linux.tar.gz | tar xvz
```

Change directory
```
cd istio-0.4.0
```

Add Istio client to PATH:
```
export PATH=$PWD/bin:$PATH
```

Deploy Istio on Kubernetes:

No mutual TLS:
```
kubectl apply -f install/kubernetes/istio.yaml
```

Mutual TLS:
```
kubectl apply -f install/kubernetes/istio-auth.yaml
```

If you get 'unable to recognize...<>...no matches' errors, run:
```
kubectl apply -f install/kubernetes/istio-customresources.yaml
```

Check that all CDRs are in place and don't have '<invalid>' reports:
```
kubectl get crd
```

If you want to automatically inject Istio sidecar to applications:
```
kubectl apply -f install/kubernetes/istio-initializer.yaml
```
> **Depending on your kube config, you might need to enable this as Alpha Feature till later releases (https://kubernetes.io/docs/admin/extensible-admission-controllers/#enable-initializers-alpha-feature)**

(Optional) Deploy Zipkin/Prometheus/ServiceGraph/Grfana:
```
kubectl apply -f install/kubernetes/addons/zipkin.yaml
kubectl apply -f install/kubernetes/addons/prometheus.yaml
kubectl apply -f install/kubernetes/addons/servicegraph.yaml
kubectl apply -f install/kubernetes/addons/grafana.yaml
```

## Load Balancer ##

Because I'm poor, workaround the fact that we ain't got no fancy cloud doohickeys, so go ahead and deploy *keepalived* that acts as VIP in a routable CIDR. More of a robust HA other than just DNS RR to each worker/HAproxy.

From this:
```
                                                  ___________________
                                                 |                   |
                                           |-----| Host IP: 10.4.0.3 |
                                           |     |___________________|
                                           |
                                           |      ___________________
                                           |     |                   |
Public ----(example.com = 10.4.0.3/4/5)----|-----| Host IP: 10.4.0.4 |
                                           |     |___________________|
                                           |
                                           |      ___________________
                                           |     |                   |
                                           |-----| Host IP: 10.4.0.5 |
                                                 |___________________|
```

To this:
```
                                               ___________________
                                              |                   |
                                              | VIP: 10.4.0.50    |
                                        |-----| Host IP: 10.4.0.3 |
                                        |     | Role: Master      |
                                        |     |___________________|
                                        |
                                        |      ___________________
                                        |     |                   |
                                        |     | VIP: Unassigned   |
Public ----(example.com = 10.4.0.50)----|-----| Host IP: 10.4.0.3 |
                                        |     | Role: Slave       |
                                        |     |___________________|
                                        |
                                        |      ___________________
                                        |     |                   |
                                        |     | VIP: Unassigned   |
                                        |-----| Host IP: 10.4.0.3 |
                                              | Role: Slave       |
                                              |___________________|
```

Create account (sa):
```
kubectl create sa kube-keepalived-vip
```

Create ClusterRole (needs READ to pods, nodes, endpoints and services:

```
echo 'apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kube-keepalived-vip
rules:
- apiGroups: [""]
  resources:
  - pods
  - nodes
  - endpoints
  - services
  - configmaps
  verbs: ["get", "list", "watch"]' | kubectl create -f -
```
  
  Bind between ClusterRole and sa:
  
```
echo 'apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kube-keepalived-vip
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-keepalived-vip
subjects:
- kind: ServiceAccount
  name: kube-keepalived-vip
  namespace: kube-system' | kubectl create -f -
```
Deploy daemon-set:

```
curl https://raw.githubusercontent.com/kubernetes/contrib/master/keepalived-vip/vip-daemonset.yaml -o /tmp/vip-daemonset.yaml
sed -i '/hostNetwork: true/a \      serviceAccount: kube-keepalived-vip' /tmp/vip-daemonset.yaml
sed -i '4a\\  namespace: kube-system' /tmp/vip-daemonset.yaml
kubectl create -f /tmp/vip-daemonset.yaml
```
**Note: the DaemonSet yaml file contains a node selector. This is not required, is just an example to show how is possible to limit the nodes where keepalived can run (otherwise just label with ```kubectl label nodes --all type=worker```**

Check that there is a 'keepalived-vip' pod running on each node.


Next, let's create an empty configmap that will contain our expose services:
```
echo "apiVersion: v1
kind: ConfigMap
metadata:
  name: vip-configmap
  namespace: kube-system
data:" | kubectl create -f -
```

Add '--cloud-provider=external' to kube-controller-manager (systemd service).

SSH into the controller/s and add ^ accordingly:
```
$ nano /etc/systemd/system/kube-controller-manager.service
$ systemctl daemon-reload
$ systemctl restart kube-controller-manager.service
```

Deploy 'keepalived-cloud-provider' (pay attention to the CIDR assigned).
Put this into a file (/tmp/keepalived-cloud-provider.yaml)
```
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  labels:
    app: keepalived-cloud-provider
  name: keepalived-cloud-provider
  namespace: kube-system
spec:
  replicas: 1
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: keepalived-cloud-provider
  strategy:
    type: RollingUpdate
  template:
    metadata:
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
        scheduler.alpha.kubernetes.io/tolerations: '[{"key":"CriticalAddonsOnly", "operator":"Exists"}]'
      labels:
        app: keepalived-cloud-provider
    spec:
      containers:
      - name: keepalived-cloud-provider
        image: quay.io/munnerz/keepalived-cloud-provider:0.0.1
        imagePullPolicy: IfNotPresent
        env:
        - name: KEEPALIVED_NAMESPACE
          value: kube-system
        - name: KEEPALIVED_CONFIG_MAP
          value: vip-configmap
        - name: KEEPALIVED_SERVICE_CIDR
          value: 192.168.2.10/29 # CIDR reserved for keepalived allocation (192.168.2.9 - 192.168.2.14)
        volumeMounts:
        - name: certs
          mountPath: /etc/ssl/certs
        resources:
          requests:
            cpu: 200m
        livenessProbe:
          httpGet:
            path: /healthz
            port: 10252
            host: 127.0.0.1
          initialDelaySeconds: 15
          timeoutSeconds: 15
          failureThreshold: 8
      volumes:
      - name: certs
        hostPath:
          path: /etc/ssl/certs
```

Deploy the sucker:
```
kubectl apply -f /tmp/keepalived-cloud-provider.yaml
```

