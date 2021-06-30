# Demonstrating GKE Dataplane-v2

## Create a GKE cluster
``` shell
gcloud container clusters create demo \
    --enable-dataplane-v2 \
    --enable-ip-alias \
    --release-channel rapid \
    --zone us-central1-c
```

## Ensure you have downloaded the kubernetes credentials file
``` shell
gcloud container clusters get-credentials demo -z us-central1-c
```

## Check the state of the system Pods:
``` shell
kubectl -n kube-system get pods -l k8s-app=cilium -o wide
```

## To get all cilium endpoints in the cluster, you could run
``` shell
kubectl -n kube-system get ciliumendpoint
```

If Dataplane V2 is running, you will see Pods with the prefix anetd- running. anetd is the networking controller for Dataplane V2.

## Network logging
Network logging at the pod level is disabled by default.

### Verify
``` shell
kubectl get NetworkLogging default -o yaml
```

## Enable network logging
Use `kubectl` to edit your configuration

```shell
kubectl edit networklogging default
```

The YAML should look something like this when you are done.
One would **NOT** normally log all allowed traffic, but it is insightful during this demo.

``` yaml
kind: NetworkLogging
apiVersion: networking.gke.io/v1alpha1
metadata:
  name: default
spec:
  cluster:
    allow:
      log: true
      delegate: false
    deny:
      log: true
      delegate: false
```
 
## Deploy demo app
```shell
kubectl create -f https://raw.githubusercontent.com/cilium/cilium/HEAD/examples/minikube/http-sw-app.yaml
kubectl get pods,svc --show-labels
```

## Check out (the lack of) network policies
From the perspective of the deathstar service, only the ships with label org=empire are allowed to connect and request landing. Since we have no rules enforced, both xwing and tiefighter will be able to request landing. To test this, use the commands below.

```shell
kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```

## Apply NetworkPolicy
```shell
kubectl apply -f net-policy.yaml
kubectl describe networkpolicy allow-empire-only
```

## Generate test traffic
Notice that the xwing connection should fail and time-out.
```shell
kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```

## Verify logs
You will have to change the `PROJECT_NAME` and related variables to something appropriate for your account.
It might be easier to use the `Logs Explorer` in the GCP Console.

```shell
gcloud logging read --project "nick-316419" 'resource.labels.cluster_name="demo" AND resource.type="k8s_node" AND logName: "projects/nick-316419/logs/policy-action"'
```

# Clean up resources
1. Delete your cluster

# References
- [GKE blog](https://cloud.google.com/blog/products/containers-kubernetes/bringing-ebpf-and-cilium-to-google-kubernetes-engine)
- [Using network policy logging](https://cloud.google.com/kubernetes-engine/docs/how-to/network-policy-logging)
- [Cilium Demo](https://docs.cilium.io/en/latest/gettingstarted/http/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [FQDN policies](https://github.com/GoogleCloudPlatform/gke-fqdnnetworkpolicies-golang)
- [Netflix and eBPF](https://netflixtechblog.com/how-netflix-uses-ebpf-flow-logs-at-scale-for-network-insight-e3ea997dca96)
