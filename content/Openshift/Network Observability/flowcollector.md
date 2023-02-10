+++
title = "Flow Collector"
date = 2023-02-09T12:04:20+01:00
weight = 4
chapter = false
+++

#### Install Network Oservability operator from Operator Hub

#### Create the following cluster role and cluster role binding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: loki-netobserv-tenant
rules:
- apiGroups:
  - 'loki.grafana.com'
  resources:
  - network
  resourceNames:
  - logs
  verbs:
  - 'get'
  - 'create'
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: loki-netobserv-tenant
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: loki-netobserv-tenant
subjects:
- kind: ServiceAccount
  name: flowlogs-pipeline              
  namespace: netobserv
- kind: ServiceAccount
  name: netobserv-plugin               
  namespace: netobserv

```

#### Create a flow collector instance

```yaml
spec:
    loki:
        authToken: FORWARD
      
        app: netobserv-flowcollector
        statusUrl: https://lokistack-netobserv-query-frontend-http.netobserv.svc:3100
        tenantID: network
        tls:
            caCert:
                certFile: service-ca.crt #Leave blank if using self-signed certificate (default)
                name: lokistack-netobserv-gateway-ca-bundle #Leave blank if using self-signed certificate (default)
                type: configmap
            enable: true
            insecureSkipVerify: true
            userCert: {}
        url: https://lokistack-netobserv-gateway-http.netobserv.svc:8080/api/logs/v1/network/
    namespace: netobserv

```

#### Once installed, Refresh the console to enable the plugin

![netobs](/images/netobs.png)