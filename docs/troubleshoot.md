# Troubleshoot section

This section may help you troubleshoot and recover from unhealthy flux states.


## Error while reconciling

Be aware that **flux helmreleases** have a count retry for reconciliation success until then they will fail in retry exceeds error.

This is define in the helmreleases files with section such as :

```yaml
  install:
    remediation:
      retries: 3
```

In this example we will allow helm controller to retry 3 times before stoping reconciling the helmrelease.
When you reach such a state, even a `flux reconcile` won't force the reconciliation.

```bash
$ flux reconcile kustomization infrastructure
► annotating Kustomization infrastructure in flux-system namespace
:heavy_check_mark: Kustomization annotated
◎ waiting for Kustomization reconciliation
✗ context deadline exceeded
```

A describe on the helmrelease explain why:

```
$ kubectl get helmrelease ingress-nginx -n ingress-nginx -o yaml 
...
**warning: Upgrade “ingress-nginx” failed: timed out waiting for the condition**
```


If you reach such a state you will need to suspend and then unsuspend the helmrelease so that the controller will try to reconcile again. For example, if you have a problem with the ingress-nginx helmrelease:

```bash
#Suspend the release
$ flux suspend helmrelease ingress-nginx -n ingress-nginx

#And resume it
$ flux resume helmrelease ingress-nginx -n ingress-nginx
```

You should end up with all Kustomizations and helm releases in a Ready state to True:

```
$ flux get kustomizations -A
NAMESPACE     NAME                  READY   STATUS                                                              AGE
flux-system   apps                  True    Applied revision: main/5a9468d098156a6a8fc07f6197ff5f76e113f75d     2d4h
flux-system   calico-installation   True    Applied revision: master/a5169d8dab9e8c4918afdf098b9c1131ef9b0400   2d4h
flux-system   calico-operator       True    Applied revision: master/a5169d8dab9e8c4918afdf098b9c1131ef9b0400   2d4h
flux-system   config                True    Applied revision: main/5a9468d098156a6a8fc07f6197ff5f76e113f75d     2d4h
flux-system   flux-system           True    Applied revision: main/5a9468d098156a6a8fc07f6197ff5f76e113f75d     2d4h
flux-system   infrastructure        True    Applied revision: main/5a9468d098156a6a8fc07f6197ff5f76e113f75d     2d4h
```
and

```
$ k get helmreleases -A
NAMESPACE        NAME                    READY   STATUS                             AGE
flagger-system   flagger                 True    Release reconciliation succeeded   2d4h
flagger-system   flagger-loadtester      True    Release reconciliation succeeded   2d4h
ingress-nginx    ingress-nginx           True    Release reconciliation succeeded   2d4h
kyverno          kyverno                 True    Release reconciliation succeeded   2d4h
monitoring       kube-prometheus-stack   True    Release reconciliation succeeded   2d4h
podinfo          podinfo                 True    Release reconciliation succeeded   2d4h
```

