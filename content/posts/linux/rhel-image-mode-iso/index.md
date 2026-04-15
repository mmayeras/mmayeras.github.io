---
title: "Build a RHEL Image Mode ISO with bootc-image-builder"
date: 2026-04-14
categories:
  - Linux
tags:
  - rhel
  - bootc
  - image-mode
summary: Build an installable ISO from a RHEL bootc container image using bootc-image-builder
toc:
  enable: true
  auto: true
---

Build an installable ISO from a [RHEL image mode](https://developers.redhat.com/products/rhel-image-mode/overview) container using [bootc-image-builder](https://github.com/osbuild/bootc-image-builder). The resulting ISO installs a system whose root filesystem is managed as an OCI container image — no traditional RPM package manager required post-install.

{{< admonition warning >}}
All placeholders in angle brackets — `<your-password>`, `<your-ssh-public-key>`, `<registry-token>`, `<your-org>`, `<server-ip>`, etc. — must be replaced with values matching the target environment before running any command or applying any manifest.
{{< /admonition >}}

## Prerequisites

- A RHEL subscription with access to `registry.redhat.io`.
- `podman` authenticated to `registry.redhat.io` (`podman login registry.redhat.io`).
- A container registry to host the bootc image (e.g. `quay.io`).
- `sudo` / root privileges on the build host.

## 1. Write the Containerfile

Define the bootc container image by extending the official **RHEL bootc** base image.

The installed system needs registry credentials to pull future updates via `bootc upgrade`. Embed the pull secret at build time using a `RUN --mount=type=secret` instruction so the credentials are never stored in a layer.

The credentials are placed in `/usr/lib/container-auth.json` (image-owned, never modified by the 3-way `/etc` merge). A `tmpfiles.d` entry recreates the symlink at `/etc/ostree/auth.json` on every boot — ensuring the link survives even if manually deleted, since `tmpfiles.d` runs from `/usr/lib` which is always authoritative.

{{< admonition note >}}
**How the 3-way merge works** ([full article](https://developers.redhat.com/articles/2025/08/25/what-image-mode-3-way-merge))

When RHEL is running in image mode and a change is made to its filesystem, a new image is created containing those changes. The system configurations that differ from the running image are merged to create a new default state. A 3-way merge incorporates a third version — older than both the current and new image — to minimize merge conflicts.

Filesystems are treated differently in image mode:

- `/usr` → **image state**: contents of the image overwrite local files
- `/etc` → **local configuration state**: contents of the image are merged with a preference for local files
- `/var` → **local state**: image contents are ignored after initial installation

This is why the auth file is placed under `/usr/lib` (image-owned, always overwritten) rather than `/etc` (where a manual deletion would be treated as a local change and preserved across upgrades).
{{< /admonition >}}

Create the tmpfiles.d config alongside the `Containerfile`:

```bash
cat > bootc-auth.conf <<'EOF'
L  /etc/ostree/auth.json  -  -  -  -  /usr/lib/container-auth.json
EOF
```

Then reference it in the `Containerfile`:

```dockerfile
FROM registry.redhat.io/rhel10/rhel-bootc:10.1

RUN --mount=type=secret,id=bootc-pull-secret,required=true \
    cp /run/secrets/bootc-pull-secret /usr/lib/container-auth.json && \
    chmod 0600 /usr/lib/container-auth.json

COPY bootc-auth.conf /usr/lib/tmpfiles.d/bootc-auth.conf
```

## 2. Build and Push the Container Image

Authenticate to both registries before building. `registry.redhat.io` is required to pull the base image; the target registry (e.g. `quay.io`) is required to push the result:

```bash
podman login registry.redhat.io
podman login quay.io
```

Generate the pull secret file that will be injected into the image at build time. This file must contain credentials for the registry from which `bootc upgrade` will pull future updates:

```bash
podman login --authfile ./bootc-pull-secret.json \
  quay.io \
  -u <registry-username> \
  --password-stdin <<< "<registry-token>"
```

{{< admonition warning >}}
Do not commit `bootc-pull-secret.json` to version control. Add it to `.gitignore`.
{{< /admonition >}}

Build the image, passing the secret file with `--secret` so it is mounted transiently during the `RUN` step and never persisted in any layer:

```bash
export IMAGE=quay.io/<your-org>/bootc:v0

podman build \
  --secret id=bootc-pull-secret,src=./bootc-pull-secret.json \
  -t ${IMAGE} .
podman push ${IMAGE}
```

The pushed image is the artifact that `bootc-image-builder` pulls to compose the ISO.

## 3. Prepare the Installer Configuration

`config.json` customizes the first-boot user created by the installer. Create it alongside the `Containerfile`:

```json
{
  "blueprint": {
    "customizations": {
      "user": [
        {
          "name": "cloud-user",
          "password": "<your-password>",
          "key": "<your-ssh-public-key>",
          "groups": ["wheel"]
        }
      ]
    }
  }
}
```

{{< admonition warning >}}
Do not commit `config.json` to version control if it contains a plaintext password. Use a secrets manager or an environment-substituted template instead.
{{< /admonition >}}

## 4. Generate the ISO

Run `bootc-image-builder` with `--type iso`. The `/var/lib/containers/storage` bind-mount lets the builder resolve the image from the host's local container storage, avoiding an extra registry pull.

Create the output directory before running the builder, otherwise the bind-mount will fail:

```bash
mkdir -p output
```

```bash
sudo podman run --rm -it --privileged \
  -v ./output:/output \
  -v ./config.json:/config.json:ro \
  -v /var/lib/containers/storage:/var/lib/containers/storage \
  --pull newer \
  registry.redhat.io/rhel10/bootc-image-builder:10.1 \
  --type iso \
  --config /config.json \
  --output /output \
  quay.io/<your-org>/bootc:v0
```

The ISO is written to `./output/` once the osbuild pipeline completes.

## Verify

```bash
ls -lh output/
```

Expected output:

```
total 1.2G
-rw-r--r--. 1 root root 1.2G Apr 14 10:00 bootimage.iso
```

## Deploy the ISO

### Option A: USB drive (bare-metal)

Write the ISO to a USB drive with `dd`:

```bash
sudo dd if=output/bootimage.iso of=/dev/sdX bs=4M status=progress oflag=sync
```

Replace `/dev/sdX` with the actual target block device. Boot the target host from the USB drive to start the installer.

### Option B: VMware datastore

Upload the ISO to a vSphere datastore using the `govc` CLI:

```bash
export GOVC_URL=https://<vcenter-fqdn>
export GOVC_USERNAME=<username>
export GOVC_PASSWORD=<password>
export GOVC_INSECURE=1  # set to 0 if using a trusted certificate

govc datastore.upload \
  -ds <datastore-name> \
  output/bootimage.iso \
  iso/bootimage.iso
```

Once uploaded, attach the ISO to a VM via the vSphere UI or `govc`:

```bash
govc vm.cdrom.insert \
  -vm <vm-name> \
  -ds <datastore-name> \
  iso/bootimage.iso
```

Boot the VM from the CD-ROM device to launch the installer.

### Option C: Kickstart

Kickstart automates the installation and is compatible with ISO, PXE, and USB boot workflows. The key difference from a standard RHEL kickstart is the `ostreecontainer` directive, which replaces the `%packages` section — the entire OS is pulled from the container image.

Create a kickstart file:

```bash
cat > bootc.ks <<'KSEOF'
%pre
mkdir -p /etc/ostree
cat > /etc/ostree/auth.json << 'EOF'
{
  "auths": {
    "quay.io": { "auth": "<base64-encoded>" }
  }
}
EOF
%end

text
network --bootproto=dhcp --device=link --activate

# Disk partitioning
clearpart --all --initlabel --disklabel=gpt
reqpart --add-boot
part / --grow --fstype xfs

# Pull the OS from the bootc container image — no %packages section needed
ostreecontainer --url quay.io/<your-org>/bootc:v0

firewall --disabled
services --enabled=sshd

# First-boot user
user --name=cloud-user --groups=wheel --plaintext --password=<your-password>
sshkey --username cloud-user "<your-ssh-public-key>"

rootpw --iscrypted locked
reboot
KSEOF
```

**Method 1 — RHEL Network Install ISO + kickstart URL:** Download the [RHEL Network Install ISO](https://access.redhat.com/downloads/content/rhel) for the target architecture. Host the kickstart file on any HTTP server:

```bash
python -m http.server 8080
```

Boot the target host from the Network Install ISO. At the boot menu, append the following to the kernel command line and press `Ctrl-X`:

```
inst.ks=http://<server-ip>:8080/bootc.ks
```

The installer fetches the kickstart over the network, pulls the container image from the registry via `ostreecontainer`, and completes the installation unattended.

**Method 2 — embed kickstart into the ISO with mkksiso:** `mkksiso` (from the `lorax` package) bakes the kickstart directly into the boot ISO so no HTTP server is required. Install `lorax` first:

```bash
sudo dnf install -y lorax
```

Embed the kickstart into the bootc ISO generated in step 4:

```bash
mkksiso --ks bootc.ks ~/Downloads/rhel-10.1-x86_64-boot.iso output/bootimage-ks.iso
```

Boot from `output/bootimage-ks.iso`. The kickstart runs automatically with no interactive input — the installer pulls the container image from the registry and completes the installation unattended. The installed system boots directly from the container image layers and can be updated atomically with `bootc upgrade`.

{{< admonition tip >}}
To serve the kickstart from an HTTP server instead of embedding it, use `--cmdline` to bake the kernel argument into the ISO.

```bash
mkksiso --cmdline "inst.ks=http://<server-ip>:8080/bootc.ks" \
  ~/Downloads/rhel-10.1-x86_64-boot.iso output/bootimage-ks.iso
```
{{< /admonition >}}

### Option D: QEMU / KVM (qcow2)

Instead of generating an ISO, `bootc-image-builder` can produce a `qcow2` disk image that can be booted directly with libvirt or QEMU — no installer pass required.

Create the output directory and run the builder with `--type qcow2`:

```bash
mkdir -p output

sudo podman run --rm -it --privileged \
  -v ./output:/output \
  -v ./config.json:/config.json:ro \
  -v /var/lib/containers/storage:/var/lib/containers/storage \
  --pull newer \
  registry.redhat.io/rhel10/bootc-image-builder:10.1 \
  --type qcow2 \
  --config /config.json \
  --output /output \
  quay.io/<your-org>/bootc:v0
```

The disk image is written to `./output/qcow2/disk.qcow2`. Boot it with `virt-install`:

```bash
virt-install \
  --name bootc-vm \
  --memory 4096 \
  --vcpus 2 \
  --disk output/qcow2/disk.qcow2 \
  --import \
  --os-variant rhel10.0
```

The VM boots directly into the installed system. Updates are applied atomically from the registry with `bootc upgrade`.

### Option E: OpenShift Virtualization (DataVolume import)

[OpenShift Virtualization](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/virtualization/index) can import the `qcow2` image produced in Option D directly into a **DataVolume** via the [Containerized Data Importer (CDI)](https://github.com/kubevirt/containerized-data-importer).

Prerequisites:

- The **OpenShift Virtualization** operator is installed.
- `oc` and `virtctl` available in `$PATH`.
- The `qcow2` image is accessible over HTTP or uploaded via `virtctl`.

**Method 1 — upload with virtctl:**

```bash
virtctl image-upload dv bootc-disk \
  --size=20Gi \
  --image-path=output/qcow2/disk.qcow2 \
  --storage-class=<storage-class> \
  --namespace=<namespace> \
  --insecure
```

**Method 2 — DataVolume manifest with HTTP source:**

Serve the qcow2 from any HTTP server first:

```bash
python -m http.server 8080 --directory output/qcow2
```

Then apply the **DataVolume**:

```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: bootc-disk
  namespace: <namespace>
spec:
  source:
    http:
      url: "http://<server-ip>:8080/disk.qcow2"
  storage:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 20Gi
    storageClassName: <storage-class>
```

Once the DataVolume reaches `Succeeded` phase, create the **VirtualMachine**:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: bootc-vm
  namespace: <namespace>
spec:
  instancetype:
    kind: VirtualMachineClusterInstancetype
    name: u1.medium
  runStrategy: RerunOnFailure
  template:
    metadata:
      labels:
        app: bootc-vm
    spec:
      architecture: amd64
      domain:
        firmware:
          bootloader:
            efi:
              secureBoot: false
        devices:
          disks:
            - name: rootdisk
              disk:
                bus: virtio
          interfaces:
            - name: default
              masquerade: {}
      networks:
        - name: default
          pod: {}
      volumes:
        - name: rootdisk
          dataVolume:
            name: bootc-disk
```

```bash
oc apply -f datavolume.yaml
oc wait dv bootc-disk --for condition=Ready --timeout=10m -n <namespace>
oc apply -f virtualmachine.yaml
```

The VM boots from the imported bootc disk. Day-2 updates are applied inside the VM with `bootc upgrade` as with any other deployment method.

### Option F: OpenShift Virtualization — ISO + Kickstart install

Rather than pre-importing a disk image, boot a VM directly from the bootc ISO (generated in step 4) and run the kickstart installer inside OpenShift Virtualization.

Upload the ISO to a DataVolume first:

```bash
virtctl image-upload dv bootc-iso \
  --size=5Gi \
  --image-path=output/bootimage.iso \
  --storage-class=<storage-class> \
  --namespace=<namespace> \
  --insecure
```

Create a blank DataVolume for the target disk:

```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: bootc-rootdisk
  namespace: <namespace>
spec:
  source:
    blank: {}
  storage:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 20Gi
    storageClassName: <storage-class>
```

Create the **VirtualMachine**, attaching the ISO as a CD-ROM and the blank PVC as the install target:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: bootc-installer
  namespace: <namespace>
spec:
  instancetype:
    kind: VirtualMachineClusterInstancetype
    name: u1.medium
  running: true
  template:
    metadata:
      labels:
        app: bootc-installer
    spec:
      architecture: amd64
      domain:
        firmware:
          bootloader:
            efi:
              secureBoot: false
        devices:
          disks:
            - name: rootdisk
              disk:
                bus: virtio
              bootOrder: 2
            - name: installiso
              cdrom:
                bus: sata
                readonly: true
              bootOrder: 1
          interfaces:
            - name: default
              masquerade: {}
      networks:
        - name: default
          pod: {}
      volumes:
        - name: rootdisk
          dataVolume:
            name: bootc-rootdisk
        - name: installiso
          dataVolume:
            name: bootc-iso
```

```bash
oc apply -f blank-dv.yaml
oc apply -f vm-installer.yaml
```

The VM boots from the ISO (`bootOrder: 1`). If using a kickstart embedded with `mkksiso` (Option C — Method 2), the installation runs unattended. If using a plain ISO, pass the kickstart URL at the boot menu via the VM console:

```
inst.ks=http://<server-ip>:8080/bootc.ks
```

Once installation completes the VM reboots from the disk (`bootOrder: 2`). Remove the ISO DataVolume and detach the CD-ROM after first boot:

```bash
oc delete dv bootc-iso -n <namespace>
```

## 5. Customize the Image

Add packages, files, or systemd units to the `Containerfile` and rebuild. The secret mount must be carried over in every build that installs or modifies files requiring registry access. For example, to install `vim` :

```dockerfile
FROM registry.redhat.io/rhel10/rhel-bootc:10.1

RUN --mount=type=secret,id=bootc-pull-secret,required=true \
    cp /run/secrets/bootc-pull-secret /usr/lib/container-auth.json && \
    chmod 0600 /usr/lib/container-auth.json

COPY bootc-auth.conf /usr/lib/tmpfiles.d/bootc-auth.conf

RUN dnf install -y vim-enhanced && dnf clean all
```

Tag the new build as a separate version to keep the previous image available as a rollback target:

```bash
export IMAGE=quay.io/<your-org>/bootc:v1

podman build \
  --secret id=bootc-pull-secret,src=./bootc-pull-secret.json \
  -t ${IMAGE} .
podman push ${IMAGE}
```

## 6. Apply the Update on a Running System

On the installed host, check whether a newer image is available at the current reference without downloading it:

```bash
sudo bootc upgrade --check
```

Pull and stage the update. The running system is not affected until reboot — updates operate in an A/B style, and the staged image is visible as `staged` in `bootc status`:

```bash
sudo bootc upgrade
```

To pull **and** immediately reboot into the new image in one step, use `--apply`:

```bash
sudo bootc upgrade --apply
```

On systems that support it, `--soft-reboot=auto` avoids a full hardware reboot when no kernel changes are queued, falling back to a regular reboot otherwise:

```bash
sudo bootc upgrade --apply --soft-reboot=auto
```

### Switch to a different tag

To switch to a different tag or image reference entirely, use `bootc switch`:

```bash
sudo bootc switch quay.io/<your-org>/bootc:v1
```

`bootc switch` stages the new image. Add `--apply` to reboot immediately after staging:

```bash
sudo bootc switch --apply quay.io/<your-org>/bootc:v1
```

See the [`bootc-upgrade` man page](https://bootc.dev/bootc/man/bootc-upgrade.8.html) for the full list of options.

### Automatic updates with bootc-fetch-apply-updates

bootc ships a systemd service and companion timer that automate the full upgrade and reboot cycle. The service is documented at [`bootc-fetch-apply-updates.service`](https://bootc.dev/bootc/man/bootc-fetch-apply-updates.service.5.html).

The service runs a single command:

```ini
# /usr/lib/systemd/system/bootc-fetch-apply-updates.service
[Unit]
Description=Apply bootc updates
Documentation=man:bootc(8)
ConditionPathExists=/run/ostree-booted

[Service]
Type=oneshot
ExecStart=/usr/bin/bootc upgrade --apply --quiet
```

The companion timer controls when it fires:

```ini
# /usr/lib/systemd/system/bootc-fetch-apply-updates.timer
[Unit]
Description=Apply bootc updates
Documentation=man:bootc(8)
ConditionPathExists=/run/ostree-booted

[Timer]
OnBootSec=1h
OnUnitInactiveSec=8h
RandomizedDelaySec=2h

[Install]
WantedBy=timers.target
```

| Timer directive | Default | Effect |
|-----------------|---------|--------|
| `OnBootSec=1h` | 1 hour | First check runs 1 hour after the system boots |
| `OnUnitInactiveSec=8h` | 8 hours | Subsequent checks run 8 hours after the last run completed |
| `RandomizedDelaySec=2h` | 2 hours | Each trigger is delayed by a random offset up to 2 hours — reduces registry load when many hosts update simultaneously |

{{< admonition tip >}}
`RandomizedDelaySec` is especially relevant in large fleets. Spreading update requests over a 2-hour window avoids thundering-herd load on the container registry.
{{< /admonition >}}

Enable and start the timer:

```bash
sudo systemctl enable --now bootc-fetch-apply-updates.timer
```

To tune the schedule without modifying the upstream unit, create a drop-in override:

```bash
sudo systemctl edit bootc-fetch-apply-updates.timer
```

Example — check every 6 hours with no random delay (suitable for a lab or small fleet):

```ini
[Timer]
# Reset inherited values from the base unit before redefining them.
# An empty assignment is required; without it systemd merges additively.
OnBootSec=
OnBootSec=30min

OnUnitInactiveSec=
OnUnitInactiveSec=6h

RandomizedDelaySec=
RandomizedDelaySec=0
```

To disable automatic updates entirely:

```bash
sudo systemctl disable --now bootc-fetch-apply-updates.timer
```

The three steps performed by the service can also be run independently:

```bash
sudo bootc upgrade --check      # check only, no download
sudo bootc upgrade              # download and stage, reboot separately
sudo bootc upgrade --apply      # download, stage, and reboot immediately
```

### Verify and rollback

Check the active and staged image at any time:

```bash
sudo bootc status
```

If the update causes issues, roll back to the previous deployment and reboot:

```bash
sudo bootc rollback
sudo systemctl reboot
```
