## Why OpenShift Is Different From Vanilla Kubernetes

Openshift designed from day:01 to run in enterprise environment where security is first priority. In Kubernetes, security is not priority and is always lenient. This cannot create security challenges esp if pod can run as root and take control of host on which it is running. Kubernetes does offer PodSecurityPolicy but it is quite easy to bypass and lack enterprise level control to restrict that pods can do and not do.

## Security Context Constraints (SCC)

### What SCC Is?
Security Context Constraints is policy which ensures pods gets the permissions what is needs, following strictly least privilege principle. SCC ensure pod do not get more rights than necessary by binding service account with SCC modes.

### Three SCC Levels
There are three level available

**Restricted Mode**: default mode for all pods. Limited permission, no host level access and sufficient to run the application

**anyuid Mode**:  Allows pods to run under any id including id 0 but still it does not provide host level access to the container which privilege mode provides. When application needs to run under specific uid, use this mode.

**Privileged** : This is like running container with root privilege. This means, container has full access to the host i.e. node on which is running. This is super privilege and should not be granted unless it is justified.

### Why GPU Operator Needs Privileged SCC

GPU operator which deploys daemon sets (pods) which needs privilege mode for the following reasons

- load kernel modules (for this you need special Linux capabilities)
- mount device files /dev as /dev/nvidia*
- HostPath Access 
	- write hooks to /etc/containers/oci/hooks.d/ directory
	- update configuration under /etc/cri-o
- HostPID, NetworkHostID and so and so forth

### How It Is Assigned — Service Account Flow
The way it works is, for every pods a service account is created and SCC permissions are binded with service account. As OpenShift implements SCC at API server level, even before the Pod is started, SCC permissions attached to the service account are checked. If they passed, pod is started. 

## CRI-O vs containerd — The Runtime Difference

### What Changes Operationally
OpenShift uses CRI-O as has runtime, which is different than plan vanilla Kubernetes (which uses containerd), if you install container toolkit using Helmchart, then directories used are difference for saving their configuration or settings

```shell
# CRI-O uses
/etc/crio/crio.conf.d
# while containerd uses
/etc/containerd/config.toml

```

### What Breaks If You Use the Wrong Path
In later case, hooks information will be stored in `config.toml`, but  before container starts CRI-O must know where the hooks are written, in this case it won't find and container cannot start or won't find libraries, devices it needs to inject into container.

## Which DaemonSets Need Privileged SCC

| DaemonSet | SCC Required | Reason |
|---|---|---|
| NVIDIA Driver | Privileged | Loads nvidia.ko kernel module |
| Container Toolkit | Privileged | Writes CRI-O hooks, /etc access |
| Device Plugin | Elevated | Mounts /dev/nvidia* into containers |
| DCGM | Elevated | GPU telemetry, perf counter access |
| GFD | Restricted | Reads node info, no host access |

## Driver Toolkit (DTK) — Air-Gapped Deployment

### What DTK Solves
NVIDIA Driver includes kernel module and  these modules must be complied against
- exact version of kernel
- matching version of kernel header
If there is mismatch, it fails.
To solve this problem, OpenShift provides a DTK as container image provides kernel header matching the kernel version of the node  and build tools to compile the nvidia driver.

### Why This Matters for DACH Industrial Customers
Without DTK, one should has internet connection from your node to download kernel modules and kernel header. This is not a standard configuration for most of the DACH industrial customer and they are running in air gapped environment

## Installation Difference — OperatorHub vs Helm

OperatorHub is OpenShift marketplace where NVIDIA upload periodically its certified GPU-Operator, you can subscribe and update the GPU-Operator based on your OpenShift version. With this configuration, cluster policy is automatically created for you. 
HelmChart may or may not contain the right GPU-Operator and hence can increases operational overhead of matching driver and maintain the lifecycle.

## Key Verification Commands

```shell

# check gpu operator pods
oc get pods -n nvidia-gpu-operator

# check cluster policy status
oc get clusterpolicy cluster-policy-name -n nvidia-gpu-operator

# Check SCC assigned to the pods
oc get scc privileged -o yaml | grep -i nvidia
oc get pods -n nvidia-gpu-operator | grep -i openshift.com/scc

# Check service account
oc get pods -n nvidia-gpu-operator -o yaml | grep -i serviceaccount

# check if the pod is privileged
oc get pods -n nvidia-gpu-operator -o yaml | grep -i privileged

```

## DACH Customer Reference — When This Conversation Comes Up

When customer ask the readiness of OpenShift environment for deploying gpu-operator, i would check the OpenShift version, SCC privilege mode can be applied to the gpu-operator's service account, as some organization will have strict policy to permit SCC privilege mode. This should be clarified before deployment begins.

Everytime during conversation comes around air-gapped environment, faster driver update and ensure compliance and stability of the environment, DTK is the right place to introduce the topic.
