# k8s-recetas

## Dashboard ##

Deploy the dashboard in the following way:

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

Expose the dashboard:
```
echo "apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kube-dash
  namespace: kube-system
spec:
  rules:
  - host: kube-dash.k8s.evilcloud.xyz
    http:
      paths:
      - backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
        path: /
" | kubectl create -f -
```
> **https://github.com/kubernetes/dashboard**

## Metrics ##

Heapster:

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/rbac/heapster-rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/standalone/heapster-controller.yaml
```

## Service Discovery ##

## Traefik ##

Set RBAC permissions:
```
kubectl apply -f https://raw.githubusercontent.com/containous/traefik/master/examples/k8s/traefik-rbac.yaml
```

Deploy DaemonSet (you can also use a pod deployment.. some pros and cons):
```
kubectl apply -f https://raw.githubusercontent.com/containous/traefik/master/examples/k8s/traefik-ds.yaml
```

Expose Traefik-ui dashboard:
```
kubectl apply -f https://gitlab.libre.nz/tomas/k8s-recetas/raw/master/traefik/web-ui.yaml
```

## Heptio stuff ##

Countour is an ingress controller leveraging Envoy:
```
kubectl create -f https://gitlab.libre.nz/tomas/k8s-recetas/raw/master/heptio/contour.yaml
```

## Istio ##
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

## Deploy Speedtest website ##

```
kubectl create -f https://raw.githubusercontent.com/ToroNZ/speedtest/docker/speedtest.yaml
```

Hit the URL specified at the bottom of ^ file... (speedtest.k8s.evilcloud.xyz ?)
