---
title: "Ansible Execution Environment Build Factory"
date: 2024-03-26
draft: false
categories:
- openshift
tags:
- openshift
#![](/posts/gitopslab/gitopslab.png)
---

<img class="special-img-class" style="width:100%" src="/posts/gitopslab/arch.png" />




### Components overview

#### Ansible Automation Platform

[Red Hat® Ansible® Automation Platform](https://www.redhat.com/en/technologies/management/ansible) is an end-to-end automation platform to configure systems, deploy software, and orchestrate advanced workflows. It includes resources to create, manage, and scale across the entire enterprise.

##### Execution Environements
These are container images which include the operating system kernel (Red Hat Enterprise Linux® Universal Base Image), automation engine (ansible-core), programming language (Python), as well as all necessary dependencies. Together, they create an isolated execution environment that can interact with—and run on—almost any IT platform.
##### Kind of EEs
- **UBI8-based**
  - [ansible-automation-platform/ee-minimal-rhel8](https://catalog.redhat.com/software/containers/ansible-automation-platform/ee-minimal-rhel8/62bd87442c0945582b2b4b37)
  - [ansible-automation-platform-24/ee-minimal-rhel8](https://catalog.redhat.com/software/containers/ansible-automation-platform-24/ee-minimal-rhel8/63a3338544b6f291781716c7)
  - [ansible-automation-platform-24/ee-supported-rhel8](https://catalog.redhat.com/software/containers/ansible-automation-platform-24/ee-supported-rhel8/63a333ce183540f5962ae01d)
  - [ansible-automation-platform-24/ee-29-rhel8](https://catalog.redhat.com/software/containers/ansible-automation-platform-24/ee-29-rhel8/63a3322ccdc6fa07ca9d7527)
- **UBI9-based**
  - [ansible-automation-platform/ee-minimal-rhel9](https://catalog.redhat.com/software/containers/ansible-automation-platform/ee-minimal-rhel9/6447df2aa123f7fc409f847e)
  - [ansible-automation-platform-24/ee-minimal-rhel9](https://catalog.redhat.com/software/containers/ansible-automation-platform-24/ee-minimal-rhel9/643d4c13fae71880450b6108)
  - [ansible-automation-platform-24/ee-supported-rhel9](https://catalog.redhat.com/software/containers/ansible-automation-platform-24/ee-supported-rhel9/643d4c7255839fe0f27b0f30)
  - [ansible-automation-platform-24/ee-29-rhel9](https://catalog.redhat.com/search?gs&q=ansible-automation-platform-24/ee-29-rhel9)


##### Ansible EE tools

- [ansible-navigator](https://ansible.readthedocs.io/projects/navigator/)
  - “Replace” ansible-playbook, ansible-doc, ansible-inventory
- [ansible-builder](https://ansible.readthedocs.io/projects/builder/en/latest/)
  - podman build wrapper
- [Podman](https://www.redhat.com/en/topics/containers/what-is-podman) / [Skopeo](https://www.redhat.com/en/topics/containers/what-is-skopeo)

##### Example Execution Environement 

```yaml
---
version: 3
dependencies:
  ansible_core:
    package_pip: ansible-core==2.14.4
  ansible_runner:
    package_pip: ansible-runner
  galaxy: 
    collections:
        - name: community.windows
        - name: ansible.utils
          version: 2.10.1
  python:
    - six
    - psutil
  system: 
    - iputils [platform:rpm]
images:
  base_image:
    name: registry.redhat.io/ansible-automation-platform/ee-minimal-rhel8

additional_build_files:
    - src: files/ansible.cfg
      dest: configs

additional_build_steps:
  prepend_galaxy:
    - ADD _build/configs/ansible.cfg ~/.ansible.cfg
  prepend_final: |
    RUN whoami
    RUN cat /etc/os-release
  append_final:
    - RUN echo This is a post-install command!
- RUN ls -la /etc
``` 


#### OpenShift GitOps

Red Hat OpenShift GitOps is an Operator that uses Argo CD as the declarative GitOps engine. It enables GitOps workflows across multicluster OpenShift and *Kubernetes* infrastructure. Using Red Hat OpenShift GitOps, administrators can consistently configure and deploy *Kubernetes*-based infrastructure and applications across clusters and development lifecycles. Red Hat OpenShift GitOps is based on the open source project   Argo CD and provides a similar set of features to what the upstream offers, with additional automation, integration into Red Hat OpenShift Container Platform and the benefits of Red Hat’s enterprise support, quality assurance and focus on enterprise security.

##### What we will deploy

<img class="special-img-class" style="width:100%" src="/posts/gitopslab/gitopsapps.png" />

#### OpenShift Pipelines

Cloud-native, continuous integration and continuous delivery (CI/CD) solution based on *Kubernetes* resources. It uses Tekton building blocks to automate deployments across multiple platforms by abstracting away the underlying implementation details. Tekton introduces a number of standard custom resource definitions (CRDs) for defining CI/CD pipelines that are portable across *Kubernetes* distributions.
Serverless CI/CD system that runs pipelines with all the required dependencies in isolated containers.
Designed for decentralized teams that work on microservice-based architecture.
Use standard CI/CD pipeline definitions that are easy to extend and integrate with the existing *Kubernetes* tools, enabling you to scale on-demand.


##### Pipeline Components

- **Task**  
  Defines a series of steps which launch specific build or delivery tools that ingest specific inputs and produce specific outputs.
- **TaskRun**
  Instantiates a Task for execution with specific inputs, outputs, and execution parameters. Can be invoked on its own or as part of a Pipeline.
- **Pipeline**  
  Defines a series of Tasks that accomplish a specific build or delivery goal. Can be triggered by an event or invoked from a PipelineRun.
- **PipelineRun**  
  Instantiates a Pipeline for execution with specific inputs, outputs, and execution parameters.
- **EventListener**  
  A Kubernetes object that listens for events at a specified port on your Kubernetes cluster. It exposes an addressable sink that receives incoming event and specifies one or more Triggers. The sink is a Kubernetes service running the sink logic inside a dedicated Pod.
- **TriggerTemplate**  
  A resource that specifies a blueprint for the resource, such as a TaskRun or PipelineRun, that you want to instantiate and/or execute when your EventListener detects an event. It exposes parameters that you can use anywhere within your resource’s template.
- **TriggerBinding**  
  Allows you to extract fields from an event payload and bind them to named parameters that can then be used in a TriggerTemplate. For instance, one can extract the commit SHA from an incoming event and pass it on to create a TaskRun that clones a repo at that particular commit.
  
##### Build Pipeline Steps

Manual Steps
1. Define execution environement specs
2. Push the code in a git repo
3. Configure a webhook to trigger the Pipeline
Automatic steps
4. The pipeline will build an image based on the EE spec
5. Push the image into quay.io
6. Configure AAP to use the image

### Deploy the LAB

{{< admonition >}}
  The installation of the Openshift Cluster is not covered in this article.
  Next steps will assume you already have a running cluster, with persistent storage available and you are logged in with cluster-admin user.
{{< /admonition >}}


The required is available in these repositories :

- [openshift-pipelines-aap-ee](https://github.com/mmayeras/openshift-pipelines-aap-ee)  
  The main repository to deploy and configure the operators
- [new-ee-demo](https://github.com/mmayeras/new-ee-demo)    
  A sample repo to store execution environement specs configured with a webhook to be built through our pipeline
- [openshift-pipelines-aap-playbooks][(https://github.com](https://github.com/mmayeras/openshift-pipelines-aap-playbooks))    
  Some playbooks example 


#### Openshift GitOps Operator Installation

1. Clone or fork [gitops-ansible-ee-demo-assets](https://github.com) 
2. Apply the kustomization file for Openshift GitOps  
```shell
$ oc apply -k 0-init/0-gitops 
```

3. Check if pods are running in openshift-gitops  
```shell
$ oc get pods -n openshift-gitops-operator
```

4. Get GitOps console route  
```shell
$ CONSOLE_URL=$(oc get route openshift-gitops-server -n openshift-gitops -o jsonpath="{'https://'}{.spec.host}{'\n'}")
```


{{< admonition >}}
  Manifests to deploy the LVM operator are available in 1-apps/0-lvm/
{{< /admonition >}}

{{< admonition warning >}}
  Secrets should not be stored as plain text in public repositories, in this example, [Sealed secrets](https://github.com/bitnami-labs/sealed-secrets) is used to store only encrypted secrets
{{< /admonition >}}

#### Optional: Sealed Secret Operator Installation

```shell
$ oc apply -k 1-apps/0-sealedsecrets/
```

{{< admonition info >}}
  Sealed secrets has to be created with your sealed secret key. This is not covered in this lab. [Refer to Sealed secrets documentation](https://github.com/bitnami-labs/sealed-secrets)
{{< /admonition >}}


#### AAP Operator installation

```shell
$ oc apply -k 1-apps/1-aap/`
```

Once deployed, you can access your Ansible Automation Controller console.

```shell
$ AAC_ROUTE=$(oc get route openshift-gitops-server -n openshift-gitops -o jsonpath="{'https://'}{.spec.host}{'\n'}")
```

You need to connect once to the controller Web UI to :
- Upload your subscrition manifest or fetch it using your Red Hat credentials
- Create an token for you admin user in order to manage resources with the Openshift Operators
  - Go to Users>admin>Tokens>Add 
  - Do not specify any Application and 'Write' as scope then click Save and copy the generated token

#### AAP Operator Integration

Create a new secret in your ansible-automation-platform namespace 

```shell
$ oc apply -k 1-apps/2-aap-integration
```

Once again, sealed secret used in this example, you can create the secret manually following these specs :

```yaml
----
apiVersion: v1
data:
  host: <AAC_ROUTE>
  token: <Previously_generated_token>
kind: Secret
metadata:
  name: admin-token
  namespace: ansible-automation-platform

type: Opaque
```

#### Openshift Pipeline installation

```shell
$ oc apply -k 1-apps/3-tekton-operator
```

#### Ansible Build EE Pipeline creation

In order to fetch packages from RH repositories, we need to use valid host entitlement, we can create a secret using the same value as etc-pki-entitlement  openshift-config-managed namespace.

It will be created using Sealed Secrets too, you can create it manually with :
```shell
$ oc get secret -n openshift-config-managed etc-pki-entitlement -o yaml | sed 's/namespace: .*/namespace: ansible-ee-build/' | oc create -f -
```

We also need an ansible config file with a valid token to fetch some collections from Red hat automation hub.

```shell
$oc -n ansible-ee-build create secret generic ansible-cfg --from-file ansible.cfg --dry-run=client -o yaml | kubeseal -o yaml | oc create -f -
```

```ini
[defaults]
host_key_checking = False

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s

[galaxy]
server_list = automation_hub, galaxy

[galaxy_server.automation_hub]
url=https://console.redhat.com/api/automation-hub/content/published/
auth_url=https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token
token=<your_token>
[galaxy_server.galaxy]
url=https://galaxy.ansible.com/
```

These two secrets are mounted in 2-tkn-ee-build-builder-task.

The github-trigger-secret is the one used to configure your custom execution-environement repository webhook to the pipeline

```shell
$oc -n ansible-ee-build create secret generic ansible-cfg --from-literal secretToken=123 --dry-run=client -o yaml | kubeseal -o yaml | oc create -f -
```

```shell
$ oc apply -k 1-apps/4-tekton-pipeline
```

You can now configure your Execution Environement repository webhook with the event listener route :

```shell
$ oc get route -n ansible-ee-build ansible-ee-el -o jsonpath="{'https://'}{.spec.host}{'\n'}"
```

and the previously created github-trigger-secret.

{{< admonition info >}}
  Sealed secrets has to be created with your sealed secret key. This is not covered in this lab. [Refer to Sealed secrets documentation](https://github.com/bitnami-labs/sealed-secrets)
{{< /admonition >}}

### Run your pipeline

See https://github.com/mmayeras/new-ee-demo.git for an example.

In this configuration, I am uploading my execution-environement images on quay.io where the destination images name will be :

 <i> quay.io/rh_ee_$(tt.params.reponame) </i>

The full reponame (configured in 2-trigger-binding.yaml) is for example mmayeras/new-ee-demo.

Each new push event will trigger the a PipelineRun and the new EE image will be push into quay.io.

Enjoy!



