# Command reference

Cluster operating commands: kubectl, flux.

## kubectl

```sh
# common kubectl commands
kubectl cluster-info                       # cluster information
kubectl get nodes                          # list nodes
kubectl get namespaces                     # list namespaces
kubectl get pods -n namespace              # list pod in namespace
kubectl get pods -A                        # list pods in all namespaces
kubectl describe node node_name            # node information
kubectl describe pod pod_name -n namespace # pod information
kubectl apply -f file_or_folder            # deploy pods
kubectl delete pods pod_name -n namespace  # delete pods
kubectl delete pods --all -n namespace     # delete all pods
kubectl logs pod_name -n namespace         # show logs of pod
kubectl logs pod_name -n namespace -f      # follow logs of pod
kubectl exec -it pod_name -- /bin/sh       # exec command in pod

# list containers inside pod
kubectl get pods -n observability -o jsonpath='{.spec.containers[*].name}' \
 kube-prometheus-stack-grafana-5db86f7cc-hqr65

# list containers inside pod using deployment name
kubectl get -n observability -o jsonpath='{.spec.template.spec.containers[*].name}' \
 deployments/kube-prometheus-stack-grafana

# enter one of the containers in the pod
kubectl exec -it -n observability deployments/kube-prometheus-stack-grafana -c grafana -- /bin/sh
```

## flux

```sh
# common flux commands
flux get all -A                    # list all resources & status
flux get kustomizations -A         # list all kustomizations & status
flux get helmreleases -A           # list all Helm releases & status
flux get sources all -A            # list all sources & status
flux get sources git -A            # list git sources & status
flux get sources helm -A           # list helm sources & status
flux check                         # controllers & CRDs healthy?
flux suspend kustomization app     # pause reconciliation (troubleshooting)
flux resume kustomization app      # resume reconciliation

# trigger a reconciliation of sources and resources
flux reconcile kustomization flux-system --with-source  # update the parent / operator first
flux reconcile kustomization app_name --with-source     # next update / build the application

# show all flux objects that are not ready
flux get all -A --status-selector ready=false

# looking for controller errors
flux logs --all-namespaces --level=error

# show flux warning events
kubectl get events -n flux-system --field-selector type=Warning

# list all sources & status with kubectl
kubectl get gitrepositories.source.toolkit.fluxcd.io -A
kubectl get helmrepositories.source.toolkit.fluxcd.io -A

# list all kustomizations or Helm releases & status with kubectl
kubectl get kustomizations.kustomize.toolkit.fluxcd.io -A
kubectl get helmreleases.helm.toolkit.fluxcd.io -A
kubectl get helmcharts.source.toolkit.fluxcd.io -A
```

## Related

- [reboot-cluster.md](reboot-cluster.md) — the reboot procedure
- [deploying-apps.md](deploying-apps.md) — the deploy/verify workflow
