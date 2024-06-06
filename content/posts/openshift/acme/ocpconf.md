---
title: "Configure OCP with letsencrypt certs"
date: 2023-06-15
tags:
- openshift
categories:
- openshift
---

### Create wildcard entries in DNS

*.cluster.domain.com

### Get acme.sh

```bash
git clone https://github.com/acmesh-official/acme.sh.git
cd acme.sh
```

### Get token from CloudFare
Cloudfare > Get CF_Zone_ID CF_Account_ID and create CF_Token with Edit Zone permission

Edit dnsapi/dns_cf.sh with these values

### Create certificates

```bash
export OCP_API_DOMAIN=$(oc whoami --show-server | cut -f 2 -d ':' | cut -f 3 -d '/' | sed 's/-api././')
export OCP_WILDCARD_DOMAIN=$(oc get ingresscontroller default -n openshift-ingress-operator -o jsonpath='{.status.domain}')
export CERTDIR=$HOME/openshift_certificates
mkdir -p ${CERTDIR}
$ ~/acme.sh/acme.sh --register-account -m your_email_address@example.com

${HOME}/acme.sh/acme.sh --issue --dns dns_cf -d ${OCP_API_DOMAIN} -d *.${OCP_WILDCARD_DOMAIN} --debug
${HOME}/acme.sh/acme.sh --install-cert -d ${OCP_API_DOMAIN} -d *.${OCP_WILDCARD_DOMAIN} --cert-file ${CERTDIR}/cert.pem --key-file ${CERTDIR}/key.pem --fullchain-file ${CERTDIR}/fullchain.pem --ca-file ${CERTDIR}/ca.cer
```

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



### Patch ingress to use custom names (if needed)
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