---
title: "Reset the OpenShift kubeadmin password"
date: 2026-06-12
categories:
  - Openshift
tags:
  - openshift
  - security
  - cluster-ops
---

Reset the **kubeadmin** bootstrap password when the original credential is lost and no other **cluster-admin** identity provider is available.


## Prerequisites

- SSH access to a control plane node (as `core`, with `sudo` if required).
- `oc` available on the node or copied in with the recovery kubeconfig.
- `htpasswd` available to generate the bcrypt hash (install `httpd-tools` on RHEL, or run from a toolbox/UBI container).
- A new password of at least 23 characters (required by the [bootstrap authenticator](https://github.com/openshift/library-go/blob/master/pkg/authentication/bootstrapauthenticator/bootstrap.go)).

{{< admonition warning "Warning" >}}
This procedure patches the **kubeadmin** secret in `kube-system`. Use only for cluster recovery. After restoring access, define an identity provider and create a dedicated **cluster-admin** user, then remove **kubeadmin** per [Red Hat documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/authentication_and_authorization/removing-kubeadmin).
{{< /admonition >}}

## 1. SSH to a control plane node

Connect to any healthy control plane node in the cluster.

```bash
ssh core@<control-plane-node>
```

## 2. Use the recovery kubeconfig

The API server exposes a localhost-only admin kubeconfig on each control plane node. Point `KUBECONFIG` at that file.

```bash
cd /etc/kubernetes/static-pod-resources/kube-apiserver-certs/secrets/node-kubeconfigs
export KUBECONFIG=localhost-recovery.kubeconfig
```

Confirm API access:

```bash
oc whoami
```

Expected output:

```text
system:admin
```

Inspect the current **kubeadmin** secret:

```bash
oc get secret -n kube-system kubeadmin -o yaml
```

The `data.kubeadmin` field holds a base64-encoded bcrypt hash of the current password.

## 3. Generate a new password hash

Choose a new password (minimum 23 characters) and produce the bcrypt hash, then base64-encode it for the secret patch.

```bash
NEW_PASS='aaaaa-bbbbb-ccccc-ddddd'
HASH_B64=$(htpasswd -bnBC 10 "" "${NEW_PASS}" | cut -c 2- | base64 -w0)
echo "${HASH_B64}"
```

The `htpasswd` output starts with `:` because no username is set; `cut -c 2-` strips that prefix before encoding.

## 4. Patch the kubeadmin secret

Update the **kubeadmin** secret with the new hash.

```bash
oc patch -n kube-system secret/kubeadmin \
  --patch "{\"data\": {\"kubeadmin\": \"${HASH_B64}\"}}"
```

Expected output:

```text
secret/kubeadmin patched
```

## 5. Log in with the new password

From any host with network access to the API, authenticate as **kubeadmin** using the new password.

```bash
oc login https://api.<cluster>.<domain>:6443 -u kubeadmin -p "${NEW_PASS}"
```

## Verify / Validate

Confirm the session and **cluster-admin** privileges:

```bash
oc whoami
oc auth can-i '*' '*' --all-namespaces
```

Expected output:

```text
kubeadmin
yes
```

Verify the secret contains the new hash:

```bash
oc get secret -n kube-system kubeadmin -o jsonpath='{.data.kubeadmin}{"\n"}'
```

The value must match `${HASH_B64}` from step 3.
