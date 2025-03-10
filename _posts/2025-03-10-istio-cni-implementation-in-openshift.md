---
title: Istio CNI Implementation in OpenShift
date: 2025-03-09 12:00:00 +0000
categories: [Kubernetes, Networking]
tags: [istio, openshift, cni, networking, service-mesh]
author: Vikas Choudhary
---

# Istio CNI Implementation in OpenShift

## Overview

Istio CNI is a component of the Istio service mesh that modifies pod networking to transparently redirect traffic through the service mesh. In OpenShift environments, Istio CNI has specific configuration and operational aspects that differ from standard Kubernetes deployments.

## Components and Deployment

Istio CNI is deployed as a daemonset named `istio-cni-node`. This daemonset runs on all cluster nodes and performs two primary functions:

1. **Deploy CNI Plugin Binary**: The container image contains the CNI plugin binary, which it copies to the node's CNI plugin directory.
2. **Generate CNI Configuration**: The daemonset creates the appropriate CNI configuration file on each node.

## Path Differences: Standard K8s vs OpenShift

| Resource | Standard K8s Path | OpenShift Path |
|----------|-------------------|----------------|
| CNI Plugin Binary | `/opt/cni/bin` | `/var/lib/cni/bin` |
| CNI Config Files | `/etc/cni/net.d/` | `/etc/cni/multus/net.d` |

## Configuration Type

The CNI configuration can be generated in two formats:

* `.conf` file (when `chained-cni=false`)
* `.conflist` file (when `chained-cni=true`)

**Important**: In OpenShift, `chained-cni` is set to `false` in the helm values when rendering the CNI daemonset. This causes the generation of a `.conf` file rather than a `.conflist` file.

## Diagram

![Istio CNI Flow Diagram](/assets/lib/istio-cni-diagram.png)

## Plugin Invocation Process

In Kubernetes, CNI plugins are invoked by container runtimes when pods are created or deleted. In OpenShift:

1. The container runtime is CRI-O
2. CRI-O scans pod annotations for the `k8s.v1.cni.cncf.io/networks` key
3. When a pod has the annotation `k8s.v1.cni.cncf.io/networks: istio-cni` (or including `istio-cni` in a comma-separated list), CRI-O recognizes it needs to invoke the Istio CNI plugin
4. CRI-O searches for a NetworkAttachmentDefinition resource:
   - If the annotation contains no namespace prefix (e.g., just `istio-cni`), it searches in the pod's namespace
   - If the annotation includes a namespace (e.g., `default/istio-cni`), it searches in that specific namespace
5. Once found, CRI-O invokes the plugin binary from `/var/lib/cni/bin/istio-cni`

## Example Pod Annotation

```yaml
metadata:
  annotations:
    k8s.v1.cni.cncf.io/networks: istio-cni
```

Alternatively, to use a NetworkAttachmentDefinition from a different namespace:

```yaml
metadata:
  annotations:
    k8s.v1.cni.cncf.io/networks: default/istio-cni
```

## NetworkAttachmentDefinition Resource

The NetworkAttachmentDefinition acts as a reference point for CRI-O to find and invoke the appropriate CNI plugin. This is created as part of the Istio CNI installation process and referenced by pod annotations.

## Customer Issue and Solution: Init Container Traffic and Istio Redirection

### Background

We encountered a customer issue in an OpenShift environment where init containers needed to bypass Istio traffic redirection since the istio-proxy container isn't ready to handle redirected traffic during init phase.

### The Problem

Prior to Istio 1.20, this was handled by running init containers with UID 1337 (the same as istio-proxy), as Istio CNI included rules to skip redirection for traffic from this UID. Beginning with Istio 1.20, this approach stopped working because:

1. Istio changed how it selects UIDs in OpenShift environments
2. Instead of using a fixed UID 1337, Istio now selects the last UID from the namespace annotation `openshift.io/sa.scc.uid-range`
3. This broke existing deployments where vault init containers were hardcoded to use UID 1337. The vault init container was hardcoded based on earlier Istio CNI rules to skip redirection for 1337 UID.
4. The customer needed a centralized solution that wouldn't require application teams to modify their deployments. Removing hard-coded UID from vault init containers will required modifying all their existing deployments.

### Solution Approaches

We considered several approaches:

1. **Using excludeOutboundIPRanges**:
   - This Istio feature skips redirection for traffic to specific IP subnets
   - Limitation: Works only for TCP traffic, not for UDP-based DNS queries
   - DNS queries would still be redirected to port 15053, failing because istio-proxy wasn't ready

2. **Setting hostAliases**:
   - Add static DNS resolution in pod specs and use excludeOutboundIPRanges
   - Rejected because it would require modifications to all application deployments
**
3. **Using k8s native sidecars for istio-proxy**
   - If we look at real cause of the problem, it is because istio-proxy is not ready to handle the redirected traffic from the init containers.
   - k8s native sidecars can solve this issue because then istio-proxy would be run as an ever running init container. You can read more [here](https://istio.io/latest/blog/2023/native-sidecars/)
   - We could not go this path because customer was not at the minimum required k8s(openshift) version for native sidecar support.

3. **Custom CNI Plugin Solution** (Selected Approach):
   - Develop a shell-based CNI plugin to add iptables rules skipping redirection for UID 1337
   - Deploy via ConfigMap and DaemonSet
   - Add NetworkAttachmentDefinition for the custom plugin
   - Modify sidecar-injection-template to include the custom plugin in network annotations

### Implementation Details

The solution uses a custom CNI plugin with these components:

1. A ConfigMap containing:
   - CNI configuration file (`99-exclude-dns.conf`)
   - Shell script implementing the CNI plugin logic

2. A DaemonSet that:
   - Mounts the ConfigMap
   - Copies the plugin to `/var/lib/cni/bin`
   - Copies the configuration to `/etc/cni/multus/net.d`

3. The CNI plugin adds an iptables rule to allow traffic from UID 1337 to bypass redirection

```json
# Example CNI configuration
{
  "kubernetes": {
    "cni_bin_dir": "/var/lib/cni/bin",
    "exclude_namespaces": [
      "istio-system",
      "kube-system"
    ]
  },
  "name": "excludedns",
  "type": "excludedns"
}
```

The DaemonSet installs this configuration and the script on all nodes, providing a centralized solution that doesn't require application teams to modify their deployments.

## Debugging Istio CNI in OpenShift

When troubleshooting Istio CNI issues in OpenShift, follow these steps:

### 1. Verify Pod Annotations

Check if the pod has the proper network annotation:

```bash
oc get pod <pod-name> -o jsonpath='{.metadata.annotations.k8s\.v1\.cni\.cncf\.io/networks}'
```

### 2. Verify NetworkAttachmentDefinition CR

Ensure the NetworkAttachmentDefinition exists in the correct namespace:

```bash
oc get network-attachment-definitions -n <namespace>
oc describe network-attachment-definition <name> -n <namespace>
```

### 3. Examine Node CNI Configuration

SSH to the node where the problematic pod is running:

```bash
oc debug node/ip-10-0-15-84.us-east-2.compute.internal
```

Once on the node, check CNI config files:

```bash
# Access host filesystem
chroot /host

# Check CNI binaries location
ls -la /var/lib/cni/bin/

# Check CNI configuration directory
ls -la /etc/cni/multus/net.d/

# Examine CNI config content
cat /etc/cni/multus/net.d/<config-file>.conf
```

### 4. Check CRI-O Logs

Look for CNI-related errors in the CRI-O logs:

```bash
journalctl -u crio | grep -i cni
```

### 5. Examine Container Details with crictl

List all containers:

```bash
crictl ps
```

Find the specific container ID for your pod:

```bash
crictl ps | grep <pod-name>
```

### 6. Determine Container PID

Get the process ID for container inspection:

```bash
crictl inspect <container-id> | grep pid
# Example output: "pid": 12345,
```

### 7. Inspect iptables Rules in Container Namespace

View the NAT table rules in the container's network namespace:

```bash
nsenter -t <pid> -n iptables -L -n -v -t nat
```

This lets you verify if the expected Istio redirection rules are present and if any custom rules (like those from the custom CNI plugin) are working as expected.

## Summary

In OpenShift environments, Istio CNI operates with specific path differences and configuration settings compared to standard Kubernetes. Understanding these details is critical both for normal operation and for implementing solutions to issues like the init container traffic redirection problem described above.

The key elements are:

* Modified filesystem paths for CNI components
* Use of `.conf` files instead of `.conflist` files
* Reliance on pod annotations and NetworkAttachmentDefinition resources for plugin invocation
* Integration with CRI-O as the container runtime
* Ability to extend with custom CNI plugins for specialized requirements
