---
title: "Cluster versions"
date: 2022-12-13
categories:
- openshift
tags:
- openshift
---

Current

`$ oc get clusterversion -o json|jq ".items[0].spec"`
```json
{
  "channel": "candidate-4.12",
  "clusterID": "1ad501e2-5e60-45a5-9890-35d56bc06a4d",
  "desiredUpdate": {
    "force": false,
    "image": "quay.io/openshift-release-dev/ocp-release@sha256:31c7741fc7bb73ff752ba43f5acf014b8fadd69196fc522241302de918066cb1",
    "version": "4.12.2"
  }
}
```

History

`$ oc get clusterversion -o json|jq ".items[0].status.history"`
```json

[
  {
    "completionTime": "2023-02-09T10:32:35Z",
    "image": "quay.io/openshift-release-dev/ocp-release@sha256:31c7741fc7bb73ff752ba43f5acf014b8fadd69196fc522241302de918066cb1",
    "startedTime": "2023-02-09T09:05:12Z",
    "state": "Completed",
    "verified": true,
    "version": "4.12.2"
  },
  {
    "completionTime": "2023-01-18T19:23:07Z",
    "image": "quay.io/openshift-release-dev/ocp-release@sha256:4c5a7e26d707780be6466ddc9591865beb2e3baa5556432d23e8d57966a2dd18",
    "startedTime": "2023-01-18T18:42:01Z",
    "state": "Completed",
    "verified": false,
    "version": "4.12.0"
  }
]
```