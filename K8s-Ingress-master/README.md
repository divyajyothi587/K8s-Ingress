# Ingress on Kubernetes with Nginx Ingress Controller and letsencrypt


## Prerequisites

In order for the Ingress resource to work, the cluster must have an ingress controller running.  Only creating an Ingress resource has no effect. You may need to deploy an Ingress controller such as `ingress-nginx`. You can choose from a number of Ingress controllers.

Unlike other types of controllers which run as part of the `kube-controller-manager` binary, Ingress controllers are not started automatically with a cluster. Use this [page](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) to choose the ingress controller implementation that best fits your cluster.


## Configure Helm charts on your k8s server if your helm client version < 3.x.x

```
kubectl create serviceaccount --namespace=kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account tiller
```


## Create an ingress controller


### How the NGINX Ingress Controller for Kubernetes Works

By default, pods of Kubernetes services are not accessible from the external network, but only by other pods within the Kubernetes cluster. Kubernetes has a built‑in configuration for HTTP load balancing, called Ingress, that defines rules for external connectivity to Kubernetes services. Users who need to provide external access to their Kubernetes services create an Ingress resource that defines rules, including the URI path, backing service name, and other information. The Ingress controller can then automatically program a frontend load balancer to enable Ingress configuration. The NGINX Ingress Controller for Kubernetes is what enables Kubernetes to configure NGINX and NGINX Plus for load balancing Kubernetes services.


```
kubectl create ns ingress-controller
helm install stable/nginx-ingress -n nginx-ingress \
    --namespace=ingress-controller \
    --set controller.replicaCount=2 \
    --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux
```


### Get the LoadBalancer IP to access Ingress controller

```
kubectl get service -l "app=nginx-ingress,component=controller" \
    --namespace=ingress-controller -o jsonpath="{.items[0].status.loadBalancer.ingress[0].hostname}"
```

**NOTE :** Add an A record / CNAME record to your DNS zone

```
*.develop.example.com  `LoadBalancer IP`
```


## Install Cert Manager

```
kubectl create ns cert-manager
# kubectl apply -f ./CRDs.yaml              ## please ignore this step, this is just for a backup. next step will download and install it for you
kubectl apply --validate=false -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.12/deploy/manifests/00-crds.yaml

helm repo add jetstack https://charts.jetstack.io
# helm install stable/cert-manager -n cert-manager --namespace=cert-manager             ## old configfuration, ignore it...!
helm install jetstack/cert-manager -n cert-manager --namespace=cert-manager
```


## Create a CA ClusterIssuer

**cluster issuer file**

Please update `email: mymail@gmail.com` part with your mail id in production. 

```
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: mymail@gmail.com
    privateKeySecretRef:
      name: letsencrypt-production
    solvers:
    - http01:
       ingress:
         class: nginx
```


```
# kubectl apply -f ./certManagerCI_staging.yaml             ## only for development / testing the conifguration , not required this for production
kubectl apply -f ./certManagerCI_production.yaml
```


## Create an Ingress Route

This Ingress route need to create oin the same `namespace` where your application : `myapp` running.

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    # nginx.ingress.kubernetes.io/rewrite-target: /$2
    cert-manager.io/cluster-issuer: letsencrypt-production
spec:
  tls:
  - hosts:
    - myapp.develop.example.com
    secretName: tls-secret
  rules:
  - host: myapp.develop.example.com
    http:
      paths:
      - backend:
          serviceName: myapp-develop
          servicePort: 80
```

### Ingress rules

Each HTTP rule contains the following information:

* An optional host. In this example, no host is specified, so the rule applies to all inbound HTTP traffic through the IP address specified. If a host is provided (for example, foo.bar.com), the rules apply to that host.

* A list of paths (for example, `/testpath`), each of which has an associated backend defined with a `serviceName` and `servicePort`. Both the host and path must match the content of an incoming request before the load balancer directs traffic to the referenced Service.

* A backend is a combination of Service and port names as described in the [Service doc](https://kubernetes.io/docs/concepts/services-networking/service/). HTTP (and HTTPS) requests to the Ingress that matches the host and path of the rule are sent to the listed backend.


## Debug Cert-manager ; You can actually find error messages in each of these, like so:

add `--namespace=YOUR_NAMESPACE` in the following command

```
kubectl get certificate
kubectl get certificaterequest
kubectl describe certificaterequest X
kubectl get order
kubectl describe order X
kubectl get challenge
kubectl describe challenge X
```


### Reference

[Configure Cert-manager with Let's Encrypt](https://cert-manager.io/docs/tutorials/acme/ingress/)

[Debug Cert-manager](https://github.com/jetstack/cert-manager/issues/2020)

[k8s-ingress](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)

[ingress-config](https://kubernetes.io/docs/concepts/services-networking/ingress/#alternatives)
