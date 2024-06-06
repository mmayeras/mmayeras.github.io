---
title: "Pod Security Admission"
date: 2022-12-13
categories:
- openshift
---


#### Pod Security Policies are deprecated in K8S 1.21

https://kubernetes.io/docs/concepts/security/pod-security-standards/

https://cloud.redhat.com/blog/pod-security-admission-in-openshift-4.11

#### New labels added in each namespace

```yaml
pod-security.kubernetes.io/enforce=privileged
pod-security.kubernetes.io/audit=restricted
pod-security.kubernetes.io/warn=restricted

security.openshift.io/scc.podSecurityLabelSync=true #Default
```

#### Specs needed for restricted profile

```yaml
securityContext:
  runAsNonRoot: true
  seccompProfile:
    type: RuntimeDefault
containers:
- name: my-cronjob-container
  securityContext:
    allowPrivilegeEscalation: false
    capabilities:
      drop: ['ALL']
```

#### Dry run apply enforce 

`$ oc label --dry-run=server --overwrite ns --all pod-security.kubernetes.io/enforce=restricted`

#### Audit script

```py
#!/usr/bin/env python3
# filename          : audit.py
# description       : Generates a list of PSA violations
# author            : mmayeras
# company           : Red Hat
# date              : 20221215
# version           : 0.1
# usage             : python audit.py [--raw, -r] [--indent, -i]
# requirements
#   Binaries:
#   - make
#   - gcc
#   Python modules:
#   - setuptools-rust
#   - openshift-client
#==============================================================================






import openshift as oc
import json
import glob
import sys, re
import argparse

def search_name(audit_line_dict):
    try:
        return audit_line_dict["responseStatus"]["details"]["name"]
    except:
        pass
    try:
        return audit_line_dict["objectRef"]["name"]
    except:
        pass

   
sys.tracebacklimit = 0
parser = argparse.ArgumentParser(description='audit psp logs')
parser.add_argument('--raw', '-r', help='only print audit logs in json format', action='store_true')
parser.add_argument('--ident', '-i', help='identation value of json output', action='store', default=4, type=int)
args = parser.parse_args()
raw = args.raw

if not raw:
    print('OpenShift client version: {}'.format(oc.get_client_version()))
    try:
        server_version = oc.get_server_version()
        print('OpenShift server version: {}'.format(server_version))
    except:
        raise ConnectionError("You must first login to your cluster.") from None

with oc.project('openshift-kube-apiserver'), oc.timeout(10*60):
    kube_apiserver_pods = oc.selector("pods", labels={"app":"openshift-kube-apiserver"} ).qnames()
    if not raw:
        print('Found the following pods in {}: {}'.format(oc.get_project_name(), kube_apiserver_pods))      
        config_map = oc.selector("cms/config").qnames() 
        for cm_obj in oc.selector(config_map[0]).objects():
            print('\nAnalyzing ConfigMap: {}'.format(cm_obj.name()))
            current_config_str = cm_obj.model.data['config.yaml']
            current_config = json.loads(current_config_str)
            admission_defaults = current_config['admission']['pluginConfig']['PodSecurity']['configuration']['defaults']
        print('\nCurrent admission controller defaults :')
        print(json.dumps(admission_defaults, indent=4))
    audit_logs = list()
    pattern = "violate(s|)"
    audit_dict = dict()
    audit_lines = list()
    for pod in kube_apiserver_pods:
        for pod_obj in oc.selector(pod).objects():
            if not raw:
                print('Analyzing pod {} logs\n'.format(pod_obj.name()))
            audit_log_files = oc.selector(pod).object().execute(['find','/var/log/kube-apiserver/','-name','audit*log'],container_name='kube-apiserver')
            for audit_log_file in list(filter(None, audit_log_files.out().split('\n'))):
                audit_logs.append(oc.selector(pod).object().execute(['grep','-hE',pattern,audit_log_file],container_name='kube-apiserver',auto_raise=False).out())
            for audit_log in list(filter(None,audit_logs)):
                audit_lines.append(audit_log.split('\n'))
                
flat_list = [item for sublist in audit_lines for item in sublist]           
for audit_line in list(filter(None,flat_list)):        
    audit_line_dict = json.loads(audit_line)

    if 'pod-security.kubernetes.io/audit-violations' not in audit_line_dict['annotations']:
        continue

    namespace = audit_line_dict["objectRef"]["namespace"]
    resource_type = audit_line_dict['objectRef']['resource']
    resource_name = search_name(audit_line_dict)
    if not resource_name:
        continue

    if audit_line_dict['objectRef']['namespace'] not in audit_dict:
        audit_dict[namespace] = dict()
    if resource_type not in audit_dict[namespace]:
        audit_dict[namespace][resource_type] = dict()
    if resource_name not in audit_dict[namespace][resource_type]:
        audit_dict[namespace][resource_type][resource_name] = list()    

    if re.sub(r'would.*?:.*?: ', '' ,audit_line_dict['annotations']['pod-security.kubernetes.io/audit-violations']).split(', ') not in audit_dict[namespace][resource_type][resource_name]:
        audit_dict[namespace][resource_type][resource_name].append(re.sub(r'would.*?:.*?: ', '' ,audit_line_dict['annotations']['pod-security.kubernetes.io/audit-violations']).split(', '))

sample_dict = "{'namespace':{'resource_type':{'resource_name':[warnings]}}}"

if len(audit_dict):
    if not raw:
        print('Found following warnings, following dict is structured like this {}:\n'.format(json.dumps(sample_dict)))
    print(json.dumps(audit_dict, indent=args.ident))
else:
    if not raw:
        print("No warnings found")
```