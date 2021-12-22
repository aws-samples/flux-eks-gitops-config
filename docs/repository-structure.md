# Repository structure

The configuration of Kubernetes clusters in this repository is structured in the following folders:

## clusters

Contains the [Kustomization](https://fluxcd.io/docs/components/kustomize/kustomization/) objects to sync with each cluster (`test` and `production`). The repository leverages [Kustomize overlays](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/#bases-and-overlays) to apply different security and application settings for `TEST` and `PRODUCTION` clusters.

## infrastructure

Contains the add-ons to be installed on the cluster as well as the [source](https://fluxcd.io/docs/components/source/) objects for the add-ons and applications installed. It's divided into two subfolders:

* **sources:** contains all the [GitRepository](https://fluxcd.io/docs/components/source/gitrepositories/) and [HelmRepository](https://fluxcd.io/docs/components/source/helmrepositories/) objects for the add-ons that will be installed.
* **add-ons:** contains the Kustomization, [HelmRelease](https://fluxcd.io/docs/components/helm/helmreleases/) and other manifests to install and configure the add-ons listed on the README.  

## config

The `config/base/policies` folder contains a set of Kyverno policies implementing a common set of security best practices. Then, there are kustomize overlays for `test` and `production` to configure Kyverno in `audit` or `enforce` mode.

These policies are meant to be used as an example and a starting point to define your own policies. This is not meant to be a comprehensive implementation of security best practices. To learn about EKS Security Best practices recommendations, please visit the EKS Best Practices guides [here](https://aws.github.io/aws-eks-best-practices/security/docs/).

## apps

Contains the applications to be deployed on the cluster. This sample repository deploys [podinfo](https://github.com/stefanprodan/podinfo) as example application and uses Kustomize overlays to set up spefic configuration to `TEST` and `PRODUCTION` clusters.

The application is configured to use progressive deployments orchestrated by Flagger and nginx ingress controller. [ServiceMonitors](https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/api.md#servicemonitor) are also configured for the Prometheus Operator to scrape application metrics and a [MetricTemplate](https://docs.flagger.app/usage/metrics#custom-metrics) to define the Canary metric to be used for our deployments. These objects are defined in `apps/base/podinfo/canary.yaml`.

```yaml
  apiVersion: flagger.app/v1beta1
  kind: Canary
  metadata:
    name: podinfo-canary
    namespace: podinfo
  spec:
    # service mesh provider can be: kubernetes, istio, appmesh, nginx, gloo
    provider: nginx
    service:
      port: 80
      targetPort: 9898
    # deployment reference
    targetRef:
      apiVersion: apps/v1
      kind: Deployment
      name: podinfo
    #Ingress reference
    ingressRef:
      apiVersion: networking.k8s.io/v1
      kind: Ingress
      name: podinfo
    # HPA reference (optional)
    autoscalerRef:
      apiVersion: autoscaling/v2beta2
      kind: HorizontalPodAutoscaler
      name: podinfo
    # the maximum time in seconds for the canary deployment
    # to make progress before rollback (default 600s)
    progressDeadlineSeconds: 60
    analysis:
      # schedule interval (default 60s)
      interval: 10s
      # max number of failed metric checks before rollback
      threshold: 10
      # max traffic percentage routed to canary
      # percentage (0-100)
      maxWeight: 50
      # canary increment step
      # percentage (0-100)
      stepWeight: 10
      # NGINX Prometheus checks
      metrics:
      - name: error-rate
        templateRef:
          name: error-rate
          namespace: podinfo
        thresholdRange:
          max: 1
        interval: 1m
      webhooks:
        - name: acceptance-test
          type: pre-rollout
          url: http://flagger-loadtester.flagger-system/
          timeout: 30s
          metadata:
            type: bash
            cmd: "curl -sd 'test' http://podinfo-canary.podinfo/token | grep token"
        - name: load-test
          url: http://flagger-loadtester.flagger-system/
          timeout: 5s
          metadata:
            cmd: "hey -z 1m -q 10 -c 2 -host podinfo.test http://myapp.example.com"
```

To test progressive deployments of podinfo go to [Flagger Canary Deployments](flagger-canary-deployments.md).

# Object Dependencies

The repository specifies dependencies between FluxCD Kustomization and HelmRelease objects as follows:

## Kustomizations

Flux starts sync'ing the infrastructure Kustomization. Once it's sync'ed and passes [health checks](https://fluxcd.io/docs/components/kustomize/kustomization/#health-assessment), it will sync the config Kustomization and finally the apps kustomization.  

```
infrastructure
      |-- config
            |-- apps
```

##  HelmReleases

Within the infrastructure Kustomization, this repo defines the following dependencies between HelmReleases:

```
[ calico | kube-prometheus-stack | metrics-server ]
                     |-------- [ kyverno | ingress-nginx ]
                                         |-------- [ flagger ]
```