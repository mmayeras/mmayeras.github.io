---
title: "Configure OCP with certs"
date: 2023-06-15
tags:
- openshift
categories:
- openshift
---

### Patch router

```yaml
oc -n openshift-ingress create secret tls router-certs --cert=${CERTDIR}/fullchain.pem --key=${CERTDIR}/key.pem
oc -n openshift-ingress-operator patch ingresscontroller default  --type=merge --patch='{"spec": { "defaultCertificate": { "name": "router-certs" }}}'
```

### Patch API cert
```yaml
oc -n openshift-config create secret tls api-certs --cert=${CERTDIR}/fullchain.pem --key=${CERTDIR}/key.pem
oc patch apiserver cluster --type merge --patch="{\"spec\": {\"servingCerts\": {\"namedCertificates\": [ { \"names\": [  \"$OCP_API_DOMAIN\"  ], \"servingCertificate\": {\"name\": \"api-certs\" }}]}}}"
```


#### Force renew if needed

```bash
export OCP_API_DOMAIN=$(oc whoami --show-server | cut -f 2 -d ':' | cut -f 3 -d '/' | sed 's/-api././')
export OCP_WILDCARD_DOMAIN=$(oc get ingresscontroller default -n openshift-ingress-operator -o jsonpath='{.status.domain}')
~/acme.sh/acme.sh --renew -d ${OCP_API_DOMAIN} -d *.${OCP_WILDCARD_DOMAIN} --force
```



### Patch ingress to use custom names
https://access.redhat.com/solutions/4853401


`oc edit ingress.config.openshift.io cluster`

```yaml
spec:
  appsDomain: custom.domain.com
  componentRoutes:
  - hostname: console.apps.domain.com
    name: console
    namespace: openshift-console
    servingCertKeyPairSecret:
      name: api-certs
  - hostname: oauth-openshift.apps.domain.com
    name: oauth-openshift
    namespace: openshift-authentication
    servingCertKeyPairSecret:
      name: api-certs
  domain: apps.domain.com
```