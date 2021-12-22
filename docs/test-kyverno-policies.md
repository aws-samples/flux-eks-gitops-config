# Test Kyverno policies

This repository defines the following cluster-level policies with Kyverno:

* Disallow host namespaces (except for calico-system namespace).
* Disallow `hostPath` volumes (except for calico-system, tigera-operator and monitoring namespaces).
* Disallow `hostPort`(except for calico-system and monitoring namespaces).
* Disallow privileged containers (except for calico namespace).
* Disallow usage of the `latest` image tag.
* Restrict the image registries images can be pulled from to corporate / defined list of repositories. To modify this list, go to `config/base/policies/restrict-image-registries.yaml`.
* Automatically create a NetworkPolicy for new namespaces that denies all ingress traffic by default (this example repository creates network policies necessary or the components that are deployed).

Please visit [Kyverno documentation](https://kyverno.io/docs/) for details on how it works and how policies are defined. Kyverno provides a large set of sample security policies [here](https://kyverno.io/policies/pod-security/), including the implementation of [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) definitions.

You can execute `kubectl get policyreport -A` to see a per namespace report of policy status:

```bash 
kubectl get policyreport -A
NAMESPACE         NAME                      PASS   FAIL   WARN   ERROR   SKIP   AGE
calico-system     polr-ns-calico-system     9      0      0      0       0      5m39s
default           polr-ns-default           0      0      0      0       0      99s
flagger-system    polr-ns-flagger-system    14     0      0      0       0      5m41s
flux-system       polr-ns-flux-system       28     0      0      0       0      5m42s
ingress-nginx     polr-ns-ingress-nginx     7      0      0      0       0      5m42s
monitoring        polr-ns-monitoring        24     0      0      0       0      5m39s
podinfo           polr-ns-podinfo           14     0      0      0       0      5m12s
tigera-operator   polr-ns-tigera-operator   5      0      0      0       0      5m40s
```

This repository uses [Kustomize overlays](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/#bases-and-overlays) defined for `TEST` and `PRODUCTION` clusters. `ClusterPolicies` are applied in `audit` mode for `TEST` clusters and `enforce` mode for `PRODUCTION` clusters.

Example `config/production/patch_validation_failure_action.yaml`

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: patch-validation-failure-action
spec:
  validationFailureAction: enforce
  background: true
```

While on audit mode, you can identify resources violating defined policies describing [policy reports](https://kyverno.io/docs/policy-reports/). To see an example in action, let's create a privileged nginx pod using the `latest` tag:

```bash
   kubectl run test --image=nginx --privileged=true         
   pod/test created
```

Now, let's describe the policy report for the default namespace. We will see a violation for the `disallow-latest-tag` policy as well as the `privileged-containers` policy:

```bash
   kubectl describe policyreport polr-ns-default
   Name:         polr-ns-default
   Namespace:    default
   Labels:       <none>
   Annotations:  <none>
   API Version:  wgpolicyk8s.io/v1alpha1
   Kind:         PolicyReport
   [...]
   Results:
     Category:  Best Practices
     Message:   validation error: An image tag is required. Rule require-image-tag failed at path /spec/containers/0/image/
     Policy:    disallow-latest-tag
     Resources:
       API Version:  v1
       Kind:         Pod
       Name:         test
       Namespace:    default
       UID:          abcf96f5-6695-4ae1-bcbe-b98e4f127157
     Rule:           require-image-tag
     Scored:         true
     Severity:       medium
     Status:         fail
     Category:       Best Practices
     Message:        validation rule 'validate-image-tag' passed.
     Policy:         disallow-latest-tag
     Resources:
       API Version:  v1
       Kind:         Pod
       Name:         test
       Namespace:    default
       UID:          abcf96f5-6695-4ae1-bcbe-b98e4f127157
     Rule:           validate-image-tag
     Scored:         true
     Severity:       medium
     Status:         pass
     Category:       Pod Security Standards (Baseline)
     Message:        validation rule 'host-path' passed.
     Policy:         disallow-host-path
     Resources:
       API Version:  v1
       Kind:         Pod
       Name:         test
       Namespace:    default
       UID:          abcf96f5-6695-4ae1-bcbe-b98e4f127157
     Rule:           host-path
     Scored:         true
     Severity:       medium
     Status:         pass
     Category:       Pod Security Standards (Baseline)
     Message:        validation rule 'host-namespaces' passed.
     Policy:         disallow-host-namespaces
     Resources:
       API Version:  v1
       Kind:         Pod
       Name:         test
       Namespace:    default
       UID:          abcf96f5-6695-4ae1-bcbe-b98e4f127157
     Rule:           host-namespaces
     Scored:         true
     Severity:       medium
     Status:         pass
     Category:       Pod Security Standards (Baseline)
     Message:        validation error: Privileged mode is disallowed. The fields spec.containers[*].securityContext.privileged and spec.initContainers[*].securityContext.privileged must not be set    to true.          . Rule priviledged-containers failed at path /spec/containers/0/securityContext/privileged/
     Policy:         disallow-privileged-containers
     Resources:
       API Version:  v1
       Kind:         Pod
       Name:         test
       Namespace:    default
       UID:          abcf96f5-6695-4ae1-bcbe-b98e4f127157
     Rule:           priviledged-containers
     Scored:         true
     Severity:       medium
     Status:         fail
     Category:       Best Practices
     Message:        validation error: Unknown image registry. Rule validate-registries failed at path /spec/containers/0/image/
     Policy:         restrict-image-registries
     Resources:
       API Version:  v1
       Kind:         Pod
       Name:         test
       Namespace:    default
       UID:          abcf96f5-6695-4ae1-bcbe-b98e4f127157
     Rule:           validate-registries
     Scored:         true
     Severity:       medium
     Status:         fail
     [...]
  Summary:
    Error:  0
    Fail:   3
    Pass:   4
    Skip:   0
    Warn:   0
  Events:   <none>
```

If your cluster is sync'ing the `PRODUCTION` configuration, Kyverno has been configured to `enforce` the policies, so you'll get an error when trying to create the pod. You can see an example below:

```bash
  kubectl run test --image=nginx --privileged=true 
  Error from server: admission webhook "validate.kyverno.svc" denied the request: 
  
  resource Pod/nginx/test was blocked due to the following policies
  
  disallow-latest-tag:
    require-image-tag: 'validation error: An image tag is required. Rule require-image-tag
      failed at path /spec/containers/0/image/'
  disallow-privileged-containers:
    priviledged-containers: 'validation error: Privileged mode is disallowed. The fields
      spec.containers[*].securityContext.privileged and spec.initContainers[*].securityContext.privileged
      must not be set to true.          . Rule priviledged-containers failed at path
      /spec/containers/0/securityContext/privileged/'
  restrict-image-registries:
    validate-registries: 'validation error: Unknown image registry. Rule validate-registries
      failed at path /spec/containers/0/image/'
```

There's also a [mutate policy](https://kyverno.io/docs/writing-policies/mutate/) configured to create a Network Policy blocking ingress traffic to all the namespaces created. To test it, run the following command to create a namespace:

```bash
kubectl create namespace test
```

Now, run the following command to see the Network policies on the namespace:

```bash
kubectl get networkpolicy -n test
```

You'll see a similar output to the following:

```yaml
apiVersion: v1
items:
- apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    creationTimestamp: "2021-11-17T14:25:30Z"
    generation: 1
    labels:
      app.kubernetes.io/managed-by: kyverno
      kyverno.io/generated-by-kind: Namespace
      kyverno.io/generated-by-name: test
      kyverno.io/generated-by-namespace: ""
      policy.kyverno.io/gr-name: gr-qz7d6
      policy.kyverno.io/policy-name: add-networkpolicy-deny-ingress
      policy.kyverno.io/synchronize: enable
    name: default-deny
    namespace: test
    resourceVersion: "68689"
    uid: c27b5426-5e55-4a02-9f0f-a2aa30d53e45
  spec:
    podSelector: {}
    policyTypes:
    - Ingress
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```

Explore the Kyverno [documentation](https://kyverno.io/docs/) to learn more about how it works. Alternatively, go to [Flagger Canary deployments](flagger-canary-deployments.md) to learn about hoe flagger enables progressive deployments in this sample. 
