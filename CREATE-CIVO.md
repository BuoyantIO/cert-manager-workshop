# `k3d` Setup for Linkerd and Cert-Manager in Production Workshop

This is the documentation - and executable code! - for creating a simple `k3d`
cluster for the Service Mesh Academy the "Linkerd and Cert-Manager in
Production" workshop. The easiest way to use this file is to execute it with
[demosh].

Things in Markdown comments are safe to ignore when reading this later. When
executing this with [demosh], things after the horizontal rule below (which is
just before a commented `@SHOW` directive) will get displayed.

[demosh]: https://github.com/BuoyantIO/demosh

We'll do nothing at all if there's already cluster named `pki`.

```bash
set -e
DEMOSH_QUIET_FAILURE=true

if kubectl --context pki config get-contexts pki >/dev/null 2>&1; then \
    echo "Cluster 'pki' already exists" >&2 ;\
    exit 1 ;\
fi
```

<!-- @import demosh/demo-tools.sh -->
----
<!-- @SHOW -->

The Linkerd and Cert-Manager in Production workshop can use pretty much any
kind of cluster, but using a Civo cluster for it can sometimes be convenient.
Here, we'll set up the one cluster that we need, but we won't install anything
yet.

First up, we need to make _absolutely certain_ that there's no context,
cluster, or user named `pki` in our KUBECONFIG. This is working around a bug
in the Civo CLI.

```bash
kubectl config delete-context pki || true
kubectl config delete-cluster pki || true
kubectl config delete-user pki || true
```

THEN we can use `civo k8s create` to create a new `pki` cluster. The only
weird bit here is that we're deliberately not installing Traefik (we don't
need it, so whatever).

```bash
civo k8s create pki --save --merge --remove-applications Traefik-v2-nodeport --wait
```

After that, make sure the new context is current...

```bash
kubectx pki
```

...and then wait until the cluster has some running pods. The `kubectl`
command in the loop will give `[]` when no pods exist, so any result with more
than 2 characters in it indicates that some pods exist. (This is obviously a
pretty basic check, but it's the way to do this without needing `jq` or the
like.)

```bash
while true; do \
    count=$(kubectl get pods -n kube-system -l k8s-app=kube-dns -o jsonpath='{ .items }' | wc -c) ;\
    if [ $count -gt 2 ]; then break; fi ;\
done
```

Finally, wait for the `kube-dns` Pod to be ready. (This is the reason that the
previous loop is there: trying to wait for a Pod that doesn't yet exist will
throw an error.)

```bash
kubectl wait pod --for=condition=ready \
        --namespace=kube-system --selector=k8s-app=kube-dns \
        --timeout=1m
```

Done! The `pki` cluster should be ready for the rest of the workshop.

<!-- @wait_clear -->
