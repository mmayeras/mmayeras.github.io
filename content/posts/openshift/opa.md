---
title: "Control allowed registries with Gatekeeper policies"
date: "2024-11-29"
tags:
- openshift
categories:
- openshift
---

[Gatekeeper Operator Overview](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.12/html/governance/gk-operator-overview#gk-operator-overview)

## Install operator

#### Create NS

```yaml
apiVersion: v1
kind: Namespace
metadata:  
  name: openshift-gatekeeper-system 
```

#### Create Subscription
```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata: 
  name: gatekeeper-operator-product
  namespace: openshift-operators 
spec:
  channel: stable
  installPlanApproval: Automatic
  name: gatekeeper-operator-product
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: gatekeeper-operator-product.v3.17.0
```

### Create OPA Gatekeeper instance
```yaml
kind: Gatekeeper
apiVersion: operator.gatekeeper.sh/v1alpha1
metadata:
  name: gatekeeper
spec:
  validatingWebhook: Enabled
```

## Create tests resources

### Mirror a test image
In this scenario I already mirrored an UBI9 image in my private repository, the image is avaialble at `quay.io/rh_ee_mmayeras/ubi9:latest``

### Create secrets to allow image pull from the private repository

Prepare a secret containing your repository pull secret

```yaml
### custom-pull-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: rh-ee-mmayeras-pull-secret
data:
  .dockerconfigjson: xxxx==
type: kubernetes.io/dockerconfigjson
```

### Create test namespaces and secrets

```yaml
apiVersion: v1
kind: Namespace
metadata:  
  name: opa-pull-allowed
```

```yaml
apiVersion: v1
kind: Namespace
metadata:  
  name: opa-pull-denied
```

  {{< admonition warning "Info" >}}
    The purpose of the tests is to demonstrate we can deny namespaces pulling from specific registries even if a pull-secret exists for these specific registries, so let's create the pull secret in both namespaces
  {{< /admonition >}}


```bash
$ oc -n opa-pull-allowed create custom-pull-secret.yaml
$ oc -n opa-pull-denied create custom-pull-secret.yaml
$ oc -n opa-pull-allowed secrets link default rh-ee-mmayeras-pull-secret  --for=pull
$ oc -n opa-pull-denied secrets link default rh-ee-mmayeras-pull-secret  --for=pull
```

### Create test deployments

Deploy the same deployment in both namespaces

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: ubi9
    app.kubernetes.io/component: ubi9
    app.kubernetes.io/instance: ubi9
  name: ubi9
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      deployment: ubi9
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        deployment: ubi9
    spec:
      containers:
      - command:
        - /bin/bash
        - -c
        - 'trap : TERM INT; sleep infinity & wait'
        image: quay.io/rh_ee_mmayeras/ubi9:latest
        imagePullPolicy: Always
        name: ubi9
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
```

```bash
$ oc create -f deploy.yaml -n opa-pull-allowed
deployment.apps/ubi9 created
$ oc create -f deploy.yaml -n opa-pull-denied
deployment.apps/ubi9 created
❯ oc -n opa-pull-allowed get pods
NAME                    READY   STATUS    RESTARTS   AGE
ubi9-77c697c8f7-zlzcq   1/1     Running   0          28s
❯ oc -n opa-pull-denied get pods
NAME                    READY   STATUS    RESTARTS   AGE
ubi9-77c697c8f7-9vfvh   1/1     Running   0          31s
```

Without any policy, as expected, the pods were able to start pulling the images using the previously created secret.
Let's scale the deployment down the deployments to 0 for now.

```bash
$ oc -n opa-pull-allowed scale deploy ubi9 --replicas 0
deployment.apps/ubi9 created
$ oc -n opa-pull-denied scale deploy ubi9 --replicas 0
deployment.apps/ubi9 created
```

Now, let's create a policy to control which registries are denied.

## Create OPA policy


  {{< admonition warning "Info" >}}
   Using template from https://github.com/open-policy-agent/gatekeeper-library
  {{< /admonition >}}

### Create ConstraintTemplate

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sdisallowedrepos
  annotations:
    metadata.gatekeeper.sh/title: "Disallowed Repositories"
    metadata.gatekeeper.sh/version: 1.0.0
    description: >-
      Disallowed container repositories that begin with a string from the specified list.
spec:
  crd:
    spec:
      names:
        kind: K8sDisallowedRepos
      validation:
        # Schema for the `parameters` field
        openAPIV3Schema:
          type: object
          properties:
            repos:
              description: The list of prefixes a container image is not allowed to have.
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sdisallowedrepos

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          image := container.image
          startswith(image, input.parameters.repos[_])
          msg := sprintf("container <%v> has an invalid image repo <%v>, disallowed repos are %v", [container.name, container.image, input.parameters.repos])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.initContainers[_]
          image := container.image
          startswith(image, input.parameters.repos[_])
          msg := sprintf("initContainer <%v> has an invalid image repo <%v>, disallowed repos are %v", [container.name, container.image, input.parameters.repos])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.ephemeralContainers[_]
          image := container.image
          startswith(image, input.parameters.repos[_])
          msg := sprintf("ephemeralContainer <%v> has an invalid image repo <%v>, disallowed repos are %v", [container.name, container.image, input.parameters.repos])
        }
```

### Create the constraint with our deny list

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sDisallowedRepos
metadata:
  name: repo-must-not-be-in-quay-rh_ee_mmayeras
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    repos:
      - "quay.io/rh_ee_mmayeras" 
```

### Test the policy

Let's scale up the deployments

```bash
$ oc -n opa-pull-allowed scale deploy ubi9 --replicas 1
deployment.apps/ubi9 created
$ oc -n opa-pull-denied scale deploy ubi9 --replicas 1
deployment.apps/ubi9 created
```

Verify no pods are created :


```bash
$ oc -n opa-pull-allowed get pods
No resources found in opa-pull-allowed namespace.
$ oc -n opa-pull-denied get pods
No resources found in opa-pull-denied namespace.
```

And list the events :

```bash
$ oc get events --sort-by .lastTimestamp -n opa-pull-denied
6s          Warning   FailedCreate        replicaset/ubi9-77c697c8f7   Error creating: admission webhook "validation.gatekeeper.sh" denied the request: [repo-must-not-be-mine] container <ubi9> has an invalid image repo <quay.io/rh_ee_mmayeras/ubi9:latest>, disallowed repos are ["quay.io/rh_ee_mmayeras"]
---
```

Now let's allow exclude the namespace opa-pull-allowed

```bash
$ oc -n openshift-gatekeeper-system patch gatekeepers.operator.gatekeeper.sh gatekeeper --patch '{"spec":{"config":{"matches":[{"excludedNamespaces":["opa-pull-allowed"],"processes":["webhook"]}]}}}' --type=merge
```

Rollout the deploy in the opa-pull-allowed 

```bash
$ oc -n opa-pull-allowed rollout restart deploy ubi9
deployment.apps/ubi9 restarted
```

The pod starts well.


Alternatively, the same can be achieved at the Deployment and/or ReplicaSet level by extending the violations list in the constraint template :


```yaml
violation[{"msg": msg}] {
        container := input.review.object.spec.template.spec.containers[_]
        image := container.image
        startswith(image, input.parameters.repos[_])
        msg := sprintf("container <%v> has an invalid image repo <%v>", [container.name, container.image])
      }
```

and the resource kinds in the constraint :


```yaml
spec:
  match:
    kinds:
    - apiGroups:
      - ""
      kinds:
      - Pod
      - ReplicationController
      - Service
    - apiGroups:
      - apps
      kinds:
      - DaemonSet
      - Deployment
      - Job
      - ReplicaSet
      - StatefulSet
```