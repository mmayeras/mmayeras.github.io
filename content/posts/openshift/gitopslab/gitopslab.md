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

- [gitops-ansible-ee-demo-assets](https://github.com)  
  The main repository to deploy and configure the operators
- [gitops-ansible-ee-demo-executionenvironment](https://github.com)    
  A sample repo to store execution environement specs configured with a webhook to be built through our pipeline
- [gitops-ansible-ee-demo-playbooks](https://github.com)    
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