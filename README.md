# Cert Manager Workshop

Welcome to our join Buoyant and Jetstack workshop on zero trust in Kubernetes with Linkerd.
We're going to walk you through installing and configuring Linkerd with a short lived
certificate. Then we're going to show you how to configure policy for a sample application
that will demonstrate the principles of zero trust.

## Requirements

* Rootful docker
* helm
* k3d
* kubectl

## Outline

* Why cryptographic identity?
* Why use cert-manager?
* Cert manager Issuers
  * CA
  * Vault
  * Venafi
  * ACME
* mTLS
* Linkerd
* Linkerd Identity
* Linkerd Policy
  * Authn vs Authz
* Workshop
* Questions

## Steps

```bash
# Load up Booksapp and Emojivoto

curl -sL run.linkerd.io/emojivoto.yml | kubectl apply -f -

kubectl create ns booksapp && \
  curl --proto '=https' --tlsv1.2 \
  -sSfL https://run.linkerd.io/booksapp.yml |
  kubectl -n booksapp apply -f -

## NOTE: All installs to be done with helm

# cert-manager install

helm repo add linkerd https://helm.linkerd.io/stable
helm repo add jetstack https://charts.jetstack.io
helm repo update

clear

## Install cert-manager

helm install cert-manager jetstack/cert-manager --namespace cert-manager \
  --create-namespace --set installCRDs=true --version v1.10.0

## Install trust-manager

helm upgrade --install -n cert-manager cert-manager-trust \
  jetstack/cert-manager-trust --wait

## Create certs for Linkerd

kubectl apply -f bootstrap_ca.yaml

## Inspect root certificate

kubectl get -n cert-manager secrets linkerd-trust-anchor -ojson |
  jq '.data."tls.crt"' -r | base64 -d | openssl x509 -noout -text

## Inspect intermediate certificate

kubectl get -n linkerd secrets linkerd-identity-issuer -ojson |
  jq '.data."tls.crt"' -r | base64 -d | openssl x509 -noout -text

# Linkerd

### Note: The Linkerd namespace was created above 
### when we installed cert-manager

## Install CRDS

helm install linkerd-crds linkerd/linkerd-crds -n linkerd

## Install Linkerd's Control Plane
## You can see we reference the already created
## CA

helm install linkerd-control-plane --namespace linkerd \
  --set identity.externalCA=true \
  --set identity.issuer.scheme=kubernetes.io/tls linkerd/linkerd-control-plane

linkerd check

## Install Linkerd Viz

helm install linkerd-viz --namespace linkerd-viz \
  --create-namespace linkerd/linkerd-viz

linkerd check

# Adding our applications
## Inject booksapp
### By inject we mean add the Linkerd proxys. This will
### enable mTLS for all our traffic and allow us to begin
### configuring policy

kubectl get deploy -n booksapp -o yaml | linkerd inject - |
  kubectl apply -f -

## Look around, this is a good time to check on your pods
## and see the current state of traffic in your cluster.

## Things mostly work

### You can confirm that with this

linkerd viz stat deploy -n booksapp

### Unfortunately while we have mTLS
### There are no effective policies

linkerd viz authz deploy -n booksapp

# Harden our ns

## Default deny

### Configure a deny policy for booksapp

kubectl annotate ns booksapp \
  config.linkerd.io/default-inbound-policy=deny

kubectl get pods -n booksapp


linkerd viz stat deploy -n booksapp

## Traffic is still there
### Apps still restart thanks to default exemptions for health checks

kubectl rollout restart -n booksapp deploy

# Now traffic is gone
## Alternately watch the traffic
# linkerd viz authz -n booksapp deployment


# linkerd viz stat deploy -n booksapp


## Allow admin traffic

### These commands will allow viz to begin talking to our
### applications. This will NOT allow any application traffic.
### We allow viz traffic first so that we can see the impact 
### of the changes we'll be making later.

kubectl apply -f manifests/booksapp/admin_server.yaml

kubectl apply -f manifests/booksapp/allow_viz.yaml

cat manifests/booksapp/admin_server.yaml

cat manifests/booksapp/allow_viz.yaml


### Allow app traffic
kubectl apply -f manifests/booksapp/authors_server.yaml

kubectl apply -f manifests/booksapp/books_server.yaml

kubectl apply -f manifests/booksapp/webapp_server.yaml

cat manifests/booksapp/authors_server.yaml

kubectl apply -f manifests/booksapp/allow_namespace.yaml

cat manifests/booksapp/allow_namespace.yaml 

### No Traffic app? no ports!
### We only created server objects for authors, books, and
### webapp because they serve traffic. our traffic generator
### only calls out to webapp on port 7000.

## At this point we've isolated our namespace and only local 
## workloads, and linkerd-viz, can speak to anything in the 
## namespace.

# Fine grained policy

## Now that we've locked down booksapp we want to further 
## isolate our applications. We'll do that using HTTPRoutes. 
## With HTTPRoutes we can specify who can do what with our 
## app.

kubectl apply -f manifests/booksapp/authors_get_route.yaml

## Wait a minute for the authors pod to become unready.
## You can safely ignore any restarts to the traffic pod.

### Why did this happen? Linkerd creates default routes for you health and readiness checks when your pods get created. This ensures you can safely and easily enforce mTLS everywhere without needing to carve out exceptions for the kubelet. When we begin creating routes Linkerd assumes we no longer want the routes it created for us.

## Lets fix our busted health checks, no more default routes

kubectl apply -f manifests/booksapp/authors_probe.yaml
## wait a minute for authors to become ready
### Check readiness

## Now that authors is ready we can enable traffic once again.

kubectl apply -f manifests/booksapp/authors_get_policy.yaml

## Check app

### Looks good but we can't update books

## Allow Webapp to create and delete authors

kubectl apply -f manifests/booksapp/authors_modify_route.yaml

kubectl apply -f manifests/booksapp/authors_modify_policy.yaml
```
