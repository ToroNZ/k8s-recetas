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
