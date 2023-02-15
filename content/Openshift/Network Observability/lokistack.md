+++
title = "Lokistack"
date = 2023-02-09T12:04:20+01:00
weight = 5
chapter = false
hidden = false
+++



```yaml
apiVersion: loki.grafana.com/v1
kind: LokiStack
metadata:
  name: lokistack-netobserv
  namespace: netobserv 
spec:
  limits:
    global:
      queries:
        queryTimeout: 1m
  managementState: Managed
  size: 1x.extra-small
  storage:
    schemas:
    - effectiveDate: "2020-10-11"
      version: v11
    secret:
      name: network-observability-s3-secret
      type: s3
  storageClassName: thin
  tenants:
    mode: openshift-network


```