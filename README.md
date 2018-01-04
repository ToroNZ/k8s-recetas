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

If you get 'uanbel to recognize... no matches' errors, run:
```
kubectl apply -f install/kubernetes/istio-customresources.yaml
```

Check that all CDRs are in place and don't have '<invalid>' reports:
```
kubectl get crd
```

If you want to automatically inject Istio sidecar to applications:
>**Might need to enable this as Alpha Feature till later releases (https://kubernetes.io/docs/admin/extensible-admission-controllers/#enable-initializers-alpha-feature)**
```
kubectl apply -f install/kubernetes/istio-initializer.yaml
```

## Load Balancer ##

Because I'm poor, workaround the fact that we ain't got no fancy cloud doohickeys, so go ahead and deploy *keepalived* that acts as LB in a routable CIDR. More of a robust HA other than just DNS RR to each worker/HAproxy.

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
echo 'apiVersion: rbac.authorization.k8s.io/v1alpha1
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
  apiVersion: rbac.authorization.k8s.io/v1alpha1
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
  namespace: default
```

An an example, let's expose the kuberentes-dashboard:
```
echo "apiVersion: v1
kind: ConfigMap
metadata:
  name: vip-configmap
data:
  192.168.2.155: kube-system/dashboard" | kubectl create -f -
```

Deploy daemon-set:

```
curl https://raw.githubusercontent.com/kubernetes/contrib/master/keepalived-vip/vip-daemonset.yaml -o /tmp/vip-daemonset.yaml
sed -i '/hostNetwork: true/a \      serviceAccount: kube-keepalived-vip' /tmp/vip-daemonset.yaml
kubectl create -f /tmp/vip-daemonset.yaml
```

