---
title: "Maintenance script"
date: 2025-03-13
draft: false
categories:
- openshift
tags:
- openshift
---

{{< admonition warning >}}
  Use at your own risk (dangerous steps commented)
{{< /admonition >}}

```bash
#!/bin/bash

set -e

USE_SSH=false
DRAINMODE=""

while getopts ":sd:" opt; do
  case ${opt} in
    s )
      USE_SSH=true
      echo "SSH mode enabled."
      ;;
    d )
      DRAINMODE=$OPTARG
      echo "Drain mode set to: $DRAINMODE"
      ;;
    \? )
      echo "Invalid option: $OPTARG" 1>&2
      echo "Usage: $0 [-s] [-d DRAINMODE]"
      echo "  -s    Use SSH to reboot nodes instead of oc debug"
      echo "  -d    Set drain mode options (e.g., --ignore-daemonsets --delete-emptydir-data --force --disable-eviction)"
      exit 1
      ;;
  esac
done

drain_node() {
    local node=$1
    echo "Draining node: $node with options: $DRAINMODE"
    #oc adm drain $node $DRAINMODE
}

reboot_node() {
    echo $USE_SSH
    local node=$1
    if $USE_SSH; then
        echo "Rebooting node using SSH: $node"
        #ssh core@$node sudo reboot
    else
        echo "Rebooting node using oc debug: $node"
        #oc debug node/$node -- chroot /host reboot
    fi
}

is_node_ready() {
    local node=$1
    oc get node $node --no-headers | awk '{print $2}'
}

nodes=$(oc get nodes -o jsonpath='{.items[*].metadata.name}')

for node in $nodes; do
    echo "Processing node: $node"

    drain_node $node

    reboot_node $node

    echo "Waiting for node $node to be ready..."
    while [[ $(is_node_ready $node) != "Ready" ]]; do
        sleep 30
    done

    echo "Node $node is ready. Proceeding with the next node..."
done

echo "All nodes have been successfully restarted."
```

