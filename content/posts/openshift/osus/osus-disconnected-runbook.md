---
title: "Disconnected OSUS on OpenShift"
date: 2026-09-15
draft: false
categories:
- openshift
tags:
- openshift
- osus
- cincinnati
- disconnected
- artifactory
---

Manifests to reproduce this lab: [osus-lab](https://github.com/mmayeras/osus-lab).

Configuring the OpenShift Update Service (OSUS) for a cluster that pulls images
through an Artifactory mirror, with the OSUS graph scrape served from a small
dedicated repo. Built and validated on a Single-Node OpenShift (SNO) lab.

Core of this is OSUS + manifest-only mirroring (§0, §2-§10). §1 (DNS) is only
needed if Artifactory doesn't already resolve in your environment — skip it if
yours does.

All hostnames, IPs and org names below are anonymized/placeholder — swap in
your own per the table below.

> **Placeholders used throughout** — replace with your values:
> - `artifactory.example.com` → your Artifactory host (`artifactory.lab.example.com`)
> - `apps.<cluster>` → your ingress wildcard domain (`apps.sno-lab.example.com`)
> - `artifactory.example.com/ocp-release-images/graph-data` → your Artifactory path for the graph-data image
> - `4.21.31` / `4.22.12` → your bridge + target versions
> - `<current-4.20.z>` → the version the cluster is on now

---

## 0. Architecture — two consumers, two paths

The single most important idea in this whole setup. There are **two different
consumers** of images, with opposite needs, and conflating them causes most of
the failures:

| Consumer | What it does | Points at | Access pattern |
|---|---|---|---|
| **Node CRI-O** (via IDMS/ITMS) | Pulls the actual upgrade payload | Artifactory proxy (existing) | By digest, no enumeration — breadth is fine |
| **OSUS graph-builder** | Reads release manifests to build the update graph | Dedicated Artifactory local repo (release images + graph-data) | Enumerates tags + fetches manifest by digest — must stay small |

**Key insight:** OSUS only reads release-image *manifests* to build graph nodes.
It never pulls the ~190-image component payload. So the OSUS repo needs
~100 MiB per version, not ~21 GiB. The payload lives once, in Artifactory, where
nodes pull it at upgrade time via IDMS.

Consequences that drive every later step:
- OSUS `releases` must point at a repo whose tag list you **control** (a *local*
  Artifactory repo you fill deliberately — never a pull-through proxy, which
  enumerates the full upstream tag history on scrape).
- Use `oc image mirror` (manifest-only copy), **not** `oc adm release mirror`
  (which always mirrors the full payload).
- OSUS bypasses IDMS/ITMS — it reads `releases` directly, so that value must be a
  path OSUS itself can resolve.

---

## 1. (Optional) DNS — make Artifactory resolve for nodes *and* pods

Skip this section if Artifactory already resolves cluster-wide. Only needed
in a lab/disconnected env with no real DNS entry for it.

Two distinct resolution paths. Fixing one does not fix the other.

- **Node / CRI-O** uses the node's `/etc/hosts` + `/etc/resolv.conf`.
- **Pods (OSUS)** resolve through cluster CoreDNS — **not** the node's `/etc/hosts`.

### 1a. Test what actually resolves

`host`/`dig`/`nslookup` query DNS directly and **ignore `/etc/hosts`** — a timeout
there is misleading. Test the way real clients resolve:

```bash
getent hosts artifactory.example.com     # respects /etc/hosts
curl -vI https://artifactory.example.com/v2/
```

### 1b. Node-level entry (covers CRI-O image pulls)

Quick, non-persistent test on a node:

```bash
oc debug node/<node>
chroot /host
echo "192.0.2.10 artifactory.example.com" >> /etc/hosts
```

Persistent (all nodes, survives reboot) via MachineConfig — idempotent systemd
oneshot, appends without clobbering the file:

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker   # duplicate with 'master' for control-plane / SNO
  name: 99-worker-artifactory-hosts
spec:
  config:
    ignition:
      version: 3.4.0
    systemd:
      units:
      - name: add-artifactory-host.service
        enabled: true
        contents: |
          [Unit]
          Description=Append Artifactory host entry to /etc/hosts
          After=network-online.target
          Wants=network-online.target
          [Service]
          Type=oneshot
          RemainAfterExit=yes
          ExecStart=/bin/bash -c "grep -q 'artifactory.example.com' /etc/hosts || echo '192.0.2.10 artifactory.example.com' >> /etc/hosts"
          [Install]
          WantedBy=multi-user.target
```

> Applying a MachineConfig drains + reboots the pool. On SNO that's your one node.

### 1c. Pod-level resolution (covers OSUS) — minimal in-cluster resolver

If external DNS is dead, pods still can't resolve Artifactory. Stand up a tiny
CoreDNS resolver **inside the cluster** and forward only the Artifactory zone to it.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lab-dns
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-lab
  namespace: lab-dns
data:
  Corefile: |
    lab.example.com:5353 {
        hosts {
            192.0.2.10 artifactory.example.com
            fallthrough
        }
        errors
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lab-dns
  namespace: lab-dns
spec:
  replicas: 2
  selector:
    matchLabels: { app: lab-dns }
  template:
    metadata:
      labels: { app: lab-dns }
    spec:
      containers:
      - name: coredns
        image: registry.redhat.io/openshift4/ose-coredns:latest
        args: ["-conf", "/etc/coredns/Corefile"]
        ports:
        - { containerPort: 5353, protocol: UDP }
        - { containerPort: 5353, protocol: TCP }
        volumeMounts:
        - { name: config, mountPath: /etc/coredns }
      volumes:
      - name: config
        configMap: { name: coredns-lab }
---
apiVersion: v1
kind: Service
metadata:
  name: lab-dns
  namespace: lab-dns
spec:
  selector: { app: lab-dns }
  ports:
  - { name: dns-udp, port: 53, targetPort: 5353, protocol: UDP }
  - { name: dns-tcp, port: 53, targetPort: 5353, protocol: TCP }
```

**Why 5353:** the restricted SCC won't let the pod bind <1024, so CoreDNS listens
on 5353 and the Service remaps 53→5353. The listen port must be set **in the
Corefile** (`zone:5353 { ... }`), not just the containerPort — mismatch gives
`bind: permission denied`.

Wire cluster DNS to it — forward **only** the Artifactory zone, to the Service
**ClusterIP** (not pod IP, not a name — avoid the resolver-resolving-itself loop):

```bash
oc get svc lab-dns -n lab-dns -o jsonpath='{.spec.clusterIP}{"\n"}'
```

```yaml
apiVersion: operator.openshift.io/v1
kind: DNS
metadata:
  name: default
spec:
  servers:
  - name: lab-artifactory
    zones:
    - lab.example.com     # this zone ONLY — never a catch-all
    forwardPlugin:
      upstreams:
      - <lab-dns-ClusterIP>               # no :port — forward assumes 53, Service serves 53
```

Verify from a pod:

```bash
oc run dnstest --image=registry.redhat.io/ubi9/ubi --rm -it --restart=Never -- \
  getent hosts artifactory.example.com
```

---

## 2. Artifactory repos

Keep the existing proxy setup for node pulls; add **one local repo** for OSUS.

- **Existing remote/proxy repos** (`quay`, `registryreedhatio`, `nvcr`, `dockerio`)
  behind a virtual repo → node pulls via IDMS/ITMS. Leave as-is.
- **New local (hosted) Docker repo** `ocp-release-images` → OSUS scrape target.
  "Local" is Artifactory's word for a repo you push into (vs. a proxy). It lives
  on the same Artifactory; the difference is it holds only what you push, so its
  tag list stays small and controlled.

> A proxy repo **cannot** serve as the OSUS `releases` target: on scrape it
> forwards the tag-list query upstream and caches the *entire* `ocp-release`
> history, recreating the memory-blowup you're avoiding.

Also confirm proxies exist for `registry.redhat.io` (OSUS operator) and
`registry.access.redhat.com` (ubi9 base, if building graph-data from Dockerfile).

---

## 3. Mirror the release images (manifest-only)

**Use `oc image mirror`, not `oc adm release mirror`.**

- `oc adm release mirror` mirrors the **full payload** (~21 GiB, 190+ component
  images) — `--to-release-image` is *additive*, it does not restrict.
- `oc image mirror` copies **just the release image** (its manifest + ~6 own
  blobs, ~100 MiB) and does not walk into the referenced components.

```bash
# authfile needs BOTH quay.io read (source) and Artifactory write (dest)
oc image mirror -a /tmp/pull-new.json \
  quay.io/openshift-release-dev/ocp-release:4.21.31-x86_64 \
  artifactory.example.com/ocp-release-images/openshift-release-dev/ocp-release:4.21.31-x86_64

oc image mirror -a /tmp/pull-new.json \
  quay.io/openshift-release-dev/ocp-release:4.22.12-x86_64 \
  artifactory.example.com/ocp-release-images/openshift-release-dev/ocp-release:4.22.12-x86_64
```

Expected output signature (good): `manifests=1`, ~`116 MiB`, single phase.
If you see `phase 1 ... manifests=190` and ~21 GiB, you used the wrong tool.

### Adding a later z-stream (e.g. once 4.22.23 ships)

Same operation, run again when a new patch becomes available — no need to
redo earlier versions, they're already in the local repo:

```bash
oc image mirror -a /tmp/pull-new.json \
  quay.io/openshift-release-dev/ocp-release:4.22.23-x86_64 \
  artifactory.example.com/ocp-release-images/openshift-release-dev/ocp-release:4.22.23-x86_64

oc image info artifactory.example.com/ocp-release-images/openshift-release-dev/ocp-release:4.22.23-x86_64
```

Refresh graph-data (§4) after adding a version — new tag alone doesn't update
channel membership/edges. Bump the `graphDataImage` tag on the UpdateService CR
(§7) to pick it up.

> **Which versions:** mirror the **path**, not a range — your current
> `<current-4.20.z>` is represented as a graph node by the graph-data image (no
> payload needed), and you mirror the **bridge (4.21.z)** + **target (4.22.z)**
> payloads. Minor jumps require stepping through each minor. Let the graph tell
> you the exact z-streams (see Appendix A).

### Verify by digest — the test that matters

OSUS fetches the manifest **by digest**. This must resolve, not just the tag:

```bash
oc image info artifactory.example.com/ocp-release-images/openshift-release-dev/ocp-release:4.21.31-x86_64
oc image info artifactory.example.com/ocp-release-images/openshift-release-dev/ocp-release:4.22.12-x86_64
```

Both must print manifest info (not `manifest unknown` / `404`).

---

## 4. Graph-data image (on Artifactory)

The graph-data image holds channel membership + edges. Build once, refresh before
each update cycle. Not per-release. Push it into the **same local repo** as the
release images (§2) — OSUS already trusts and authenticates to that repo, so no
extra CA/credential wiring is needed.

`Dockerfile`:

```dockerfile
FROM registry.access.redhat.com/ubi9/ubi:latest
RUN curl -L -o cincinnati-graph-data.tar.gz https://api.openshift.com/api/upgrades_info/graph-data
RUN mkdir -p /var/lib/cincinnati-graph-data && \
    tar xvzf cincinnati-graph-data.tar.gz -C /var/lib/cincinnati-graph-data/ --no-overwrite-dir --no-same-owner
CMD ["/bin/bash", "-c", "exec cp -rp /var/lib/cincinnati-graph-data/* /var/lib/cincinnati/graph-data"]
```

```bash
podman build -f ./Dockerfile -t artifactory.example.com/ocp-release-images/graph-data:latest .
podman push artifactory.example.com/ocp-release-images/graph-data:latest
```

> Tag each refresh (e.g. `:2026-09-15`) and bump the CR, rather than reusing
> `:latest` — the OSUS init container loads graph-data at pod start and won't pick
> up a re-pushed `:latest` on its own.

---

## 5. Trust — the Artifactory CA chain

Artifactory presents a **leaf-only** handshake signed by an intermediate, whose
issuer is a private root. OSUS trusts neither by default. Build a bundle with
**intermediate + root** under the key OSUS requires: `updateservice-registry`.

Diagnose what the server sends and its chain:

```bash
openssl s_client -connect artifactory.example.com:443 -showcerts </dev/null 2>/dev/null \
  | openssl x509 -noout -issuer -subject
# subject = leaf (the host), issuer = the intermediate → you need intermediate + its root
```

ConfigMap (in `openshift-config`), both certs under the one key:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: artifactory-registry-ca
  namespace: openshift-config
data:
  updateservice-registry: |
    -----BEGIN CERTIFICATE-----
    <intermediate: 2023 Certificate Authority RHCSv2>
    -----END CERTIFICATE-----
    -----BEGIN CERTIFICATE-----
    <root: Internal Root CA (self-signed)>
    -----END CERTIFICATE-----
```

Validate the chain **locally** before applying (fast feedback vs. pod logs):

```bash
# leaf.crt = server cert; bundle.pem = intermediate + root
openssl s_client -connect artifactory.example.com:443 </dev/null 2>/dev/null \
  | openssl x509 -out leaf.crt
openssl verify -CAfile bundle.pem leaf.crt      # want: leaf.crt: OK
```

Wire it into the cluster image trust (OSUS operator reads
`updateservice-registry` from here):

```bash
oc patch image.config.openshift.io/cluster --type=merge \
  -p '{"spec":{"additionalTrustedCA":{"name":"artifactory-registry-ca"}}}'
```

> The benign log line `unable to process certificate ca-bundle.trust.crt ...
> Expecting: CERTIFICATE` is noise — ignore it. The real errors are
> `certificate verify failed` (missing CA in bundle) or `unable to get issuer
> certificate` (missing root above the intermediate).

---

## 6. Credentials

OSUS mounts the **global cluster pull secret** at
`/var/lib/cincinnati/registry-credentials/.dockerconfigjson`. It needs an
Artifactory entry — which your nodes already use, so it likely exists. Confirm:

```bash
oc get secret pull-secret -n openshift-config \
  -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq '.auths | keys'
# expect artifactory.example.com in the list
```

If missing, add it:

```bash
oc get secret pull-secret -n openshift-config \
  -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d > /tmp/pull.json
oc registry login --registry=artifactory.example.com \
  --auth-basic="<user>:<pass>" --to=/tmp/pull.json
oc set data secret/pull-secret -n openshift-config \
  --from-file=.dockerconfigjson=/tmp/pull.json
```

> Updating the global pull secret triggers a node rollout (SNO reboots). Wait for
> `oc get mcp` to settle.

---

## 7. Install OSUS + create the UpdateService

Install the **OpenShift Update Service** operator into
`openshift-update-service` (OperatorHub, or Subscription; package
`cincinnati-operator`). Then:

```yaml
apiVersion: updateservice.operator.openshift.io/v1
kind: UpdateService
metadata:
  name: service
  namespace: openshift-update-service
spec:
  replicas: 2
  releases: artifactory.example.com/ocp-release-images/openshift-release-dev/ocp-release
  graphDataImage: artifactory.example.com/ocp-release-images/graph-data:latest
```

Restart the graph-builder and watch for success:

```bash
oc rollout restart deployment/service -n openshift-update-service
oc logs -f deployment/service -n openshift-update-service -c graph-builder | grep -E 'valid releases|error'
```

Target line: `graph update completed, N valid releases`.

> **N will be larger than the number you mirrored** (e.g. 54): graph-data knows
> all channel versions as *nodes*. Only the versions you actually mirrored are
> installable — edges to unmirrored versions will stall at pull time.

Confirm the graph serves your versions:

```bash
PE=$(oc -n openshift-update-service get updateservice service -o jsonpath='{.status.policyEngineURI}')
curl -sk "$PE/api/upgrades_info/v1/graph?channel=stable-4.22" | jq '.nodes[].version'
```

---

## 8. Point the CVO at OSUS (via the Route, not the service)

The CVO is **host-networked** and resolves via the node's resolver — it cannot
resolve `.svc.cluster.local`. Use the OSUS **Route** URI:

```bash
PE=$(oc -n openshift-update-service get updateservice service -o jsonpath='{.status.policyEngineURI}')
echo "$PE"    # must be https://...apps.<cluster>...

oc adm upgrade channel stable-4.22
oc patch clusterversion version --type merge \
  -p "{\"spec\":{\"upstream\":\"${PE}/api/upgrades_info/v1/graph\"}}"
```

Ensure `*.apps.<cluster>` resolves from the node (same DNS work as §1), or the
CVO gets `no such host`.

---

## 9. Trust the OSUS Route cert (CVO → Route)

Console error `Unable to retrieve available updates: ... certificate signed by
unknown authority` means the CVO doesn't trust the Route's TLS cert. On SNO the
Route uses the ingress operator's **self-signed wildcard CA**.

Identify the issuer:

```bash
oc get secret router-certs-default -n openshift-ingress \
  -o jsonpath='{.data.tls\.crt}' | base64 -d \
  | openssl x509 -noout -issuer -subject
# issuer=CN=ingress-operator@<ts>, subject=CN=*.apps.<cluster>  → self-signed ingress CA
```

Grab that CA from the managed configmap and add it to the **cluster-wide proxy
trust bundle** (where the CVO looks):

```bash
oc get configmap default-ingress-cert -n openshift-config-managed \
  -o jsonpath='{.data.ca-bundle\.crt}' > /tmp/ingress-ca.crt

oc create configmap osus-route-ca -n openshift-config \
  --from-file=ca-bundle.crt=/tmp/ingress-ca.crt        # key MUST be ca-bundle.crt

oc patch proxy/cluster --type merge \
  -p '{"spec":{"trustedCA":{"name":"osus-route-ca"}}}'
```

> Triggers a node reconcile — wait for `oc get co` to settle. The ingress CA CN
> carries a timestamp and rotates on expiry; if the cluster is long-lived, replace
> the default wildcard with a cert from a stable CA instead of pinning this one.

---

## 10. Verify end to end

```bash
oc adm upgrade
oc get clusterversion -o jsonpath='{.status.conditions}' | jq   # no RemoteFailed
```

You should see the path toward 4.22.12 sourced from your own OSUS.

**Before an actual upgrade**, confirm your node-side Artifactory IDMS covers
**both** release paths the CVO pulls — this is the silent failure where OSUS
recommends an upgrade the cluster then can't pull:

- `quay.io/openshift-release-dev/ocp-release`
- `quay.io/openshift-release-dev/ocp-v4.0-art-dev`  ← the components; easy to miss

```bash
oc get idms -o yaml | grep -A2 'ocp-v4.0-art-dev'    # must be present
```

---

## Appendix A — Read the upgrade path off the graph

Before mirroring, find the exact z-streams on your path (don't guess):

```bash
# current version
oc get clusterversion -o jsonpath='{.status.desired.version}{"\n"}'

# reachable versions in the target channel
curl -s "https://api.openshift.com/api/upgrades_info/v1/graph?channel=stable-4.22" \
  | jq -r '.nodes[].version' | sort -V | grep -E '^4\.(21|22)\.'
```

Optionally let oc-mirror compute the path with `shortestPath: true` in an
ImageSetConfiguration (min = current, max = target, channel = target).

---

## Appendix B — Extract live manifests from your own cluster

Where a template is fiddlier than reading your actual state, pull the real object:

```bash
# current UpdateService CR
oc get updateservice service -n openshift-update-service -o yaml

# DNS operator config (check zone scoping + forward upstreams)
oc get dns.operator/default -o yaml

# image trust reference
oc get image.config.openshift.io/cluster -o yaml

# proxy trust bundle reference
oc get proxy/cluster -o yaml

# what release-images tags exist
oc get istag -n openshift 2>/dev/null | grep -E 'release'   # if you used internal registry
# or query Artifactory directly:
curl -s https://artifactory.example.com/artifactory/api/docker/ocp-release-images/v2/openshift-release-dev/ocp-release/tags/list | jq

# confirm OSUS SA can read a namespace's images (if cross-namespace pull)
oc auth can-i get imagestreams/layers -n openshift \
  --as=system:serviceaccount:openshift-update-service:default
```

