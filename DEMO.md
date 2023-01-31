# Linkerd and Cert-Manager

This is the documentation - and executable code! - for the "Linkerd and
Cert-Manager in Production" workshop. The easiest way to use this file is to
execute it with [demosh].

Things in Markdown comments are safe to ignore when reading this later. When
executing this with [demosh], things after the horizontal rule below (which
is just before a commented `@SHOW` directive) will get displayed.

[demosh]: https://github.com/BuoyantIO/demosh

This workshop requires that you have a running Kubernetes cluster. The README
is written assuming that you're using a local cluster that has ports exposed
to the local network: you can use [CREATE.md](CREATE.md) to set up a `k3d`
cluster that will work well. If you want to use some other kind of cluster,
make sure that the DNS is set up, and modify the DNS names to match for the
rest of the instructions.

<!-- @import demosh/demo-tools.sh -->
<!-- @import demosh/check-requirements.sh -->
<!-- @start_livecast -->
---
<!-- @SHOW -->

In this workshop, we will demonstrate how to use `cert-manager` to manage
Linkerd's trust certificates. We'll start with a self-signed root certificate,
then show how easy it is to switch such a setup to use an existing corporate
CA.

All the manifests used in the demo are in the `manifests` directory. We'll use
Helm to install everything.

First things first: make sure that everything is set up for the workshop.

```bash
#@$SHELL CREATE-CIVO.md
```

OK, everything should be ready to go!

<!-- @wait_clear -->

## Helm setup

We're going to use Helm to install three separate tools:

- `linkerd` is, of course, our favorite service mesh
- `cert-manager` does cool stuff with certificates
- `trust-manager` does cool stuff with trust

First let's tackle Helm repos -- one for Linkerd, one for Jetstack tools.

```bash
helm repo add linkerd https://helm.linkerd.io/stable
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

<!-- @wait_clear -->

## cert-manager and trust-manager

Once our repos are set up, we'll install `cert-manager`. `cert-manager` comes
first because it's going to wrangle the certificates we use for Linkerd.

```bash
helm install cert-manager jetstack/cert-manager \
     --namespace cert-manager --create-namespace \
     --set installCRDs=true --version v1.10.0 \
     --wait
```

Next up: `trust-manager`, so that we can use it to provide Linkerd with access
to the certificates that `cert-manager` creates.

```bash
helm upgrade --install cert-manager-trust jetstack/cert-manager-trust \
     --namespace cert-manager \
     --wait
```

<!-- @wait_clear -->

## `cert-manager`: Self-Signed Root

We're going to start by demonstrating how to use `cert-manager` to manage
Linkerd's certificates in a completely self-contained way.

1. We'll use cert-manager to create a self-signed root CA. In the real world
   you'd likely already have some established corporate CA to use, but again,
   this is the completely self-contained demo.

<!-- @wait -->

2. We'll use that root CA to create a trust anchor certificate which is signed
   by the root CA. This will be stored in the `cert-manager` namespace, in the
   `linkerd-identity-trust-roots` Secret.

   (It's important to note that Linkerd will not have access to anything in
    the `cert-manager` namespace. In fact, nothing but `cert-manager` should
    be able to access the `cert-manager` namespace -- set your RBAC
    accordingly!)

<!-- @wait -->

3. We'll use that trust anchor to set up _another_ CA, which we'll use to sign
   our issuer certificate. The issuer certificate must be stored in the
   `linkerd` namespace, because Linkerd needs to use it to sign workload
   certificates.

<!-- @wait -->

4. Finally, we'll use `trust-manager` to copy the public key for our trust
   anchor from the `cert-manager` namespace to the `linkerd` namespace, so
   that Linkerd has access to it.

<!-- @wait -->

The payoff here is that our trust anchor is _not_ self-signed in this setup -
which means that cert-manager can handle rotating it! - and its private key is
_never_ stored anywhere that Linkerd can see it.

<!-- @wait_clear -->

## Create the Linkerd namespace

Off we go. Remember that we'll use `cert-manager` to create certificates in
its own `cert-manager` namespace, then we'll use `trust-manager` to copy
things into the `linkerd` namespace. That means that our first job has to be
to create the `linkerd` namespace:

```bash
kubectl create ns linkerd
```

<!-- @wait_clear -->

## Set up the self-signed root CA and the trust anchor

Now that we have the namespaces we need, let's set up `cert-manager`'s root
CA, and use it to create a Linkerd trust anchor. The root CA is really simple:

```bash
less manifests/cert-manager-root-ca.yaml
```

Remember: in real production, this self-signed root CA is **not** what you'd
want to do. A really nice thing about `cert-manager`, though, is that it's
easy to switch the root CA for a different type.

For now, let's go ahead and set up the self-signed root CA:

```bash
kubectl apply -f manifests/cert-manager-root-ca.yaml
```

Once that's done, we can tell `cert-manager` how to create a trust anchor for
Linkerd:

```bash
less manifests/cert-manager-trust-anchor.yaml
```

Let's set that up too:

```bash
kubectl apply -f manifests/cert-manager-trust-anchor.yaml
```

<!-- @wait_clear -->

At this point, `cert-manager` should have created the trust anchor for us:

```bash
kubectl describe secret -n cert-manager linkerd-identity-trust-roots
```

Just to prove that there's really a certificate in there, let's take a closer
look at it. See the `ca.crt` key listed in the output? Its value is a
base64-encoded certificate (which, yes, means that it's base64-encoded
base64-encoded data -- oh well).

If we unwrap one layer of base64 encoding, we can feed that field into `step
certificate inspect`:

```bash
kubectl get secret -n cert-manager linkerd-identity-trust-roots \
        -o jsonpath='{ .data.tls\.crt }' \
        | base64 -d \
        | step certificate inspect
```

<!-- @wait_clear -->

## Set up the identity issuer certificate

Onward! We have a trust anchor certificate and a corresponding CA, so we can
tell `cert-manager` how to create the identity issuer certificate as well. A
very important note here: the identity issuer certificate must be created in
the `linkerd` namespace, so that Linkerd can use it to issue workload
certificates.

Here's the YAML:

```bash
less manifests/cert-manager-identity-issuer.yaml
```

which we will go ahead and apply:

```bash
kubectl apply -f manifests/cert-manager-identity-issuer.yaml
```

<!-- @wait_clear -->

At this point, `cert-manager` should have created the identity issuer for us:

```bash
kubectl get secrets -n linkerd
```

And, once again, we can look at the details of the identity issuer
certificate:

```bash
kubectl get secret -n linkerd linkerd-identity-issuer \
        -o jsonpath='{ .data.tls\.crt }' \
        | base64 -d \
        | step certificate inspect
```

<!-- @wait_clear -->

## Set up `trust-manager` to let Linkerd see the trust anchor

Finally, we'll use `trust-manager` to copy the trust anchor's public key into
the trust anchor bundle ConfigMap that Linkerd uses. Here's how we set that
up:

```bash
less manifests/trust-manager-ca-bundle.yaml
```

We can apply that...

```bash
kubectl apply -f manifests/trust-manager-ca-bundle.yaml
```

...and we should see the ConfigMap after that.

```bash
kubectl get cm -n linkerd
```

<!-- @wait_clear -->

## Installing Linkerd

After all that, we can install Linkerd! First up, we'll use Helm to install
the CRDs (remember that the Namespace already exists):

```bash
helm install linkerd-crds linkerd/linkerd-crds --namespace linkerd
```

After that, install the control plane, and make sure to tell it to use the
external CA that `cert-manager` is managing.

```bash
helm install linkerd-control-plane linkerd/linkerd-control-plane \
    --namespace linkerd \
    --set identity.externalCA=true \
    --set identity.issuer.scheme=kubernetes.io/tls
```

At this point, we can use `linkerd check` to wait for everything to be
working, paying extra attention to the certificate checks!

```bash
linkerd check
```

<!-- @wait_clear -->

