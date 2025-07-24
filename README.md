# GPU Driver Container
This repository contains build configurations to build a driver container for
the NVIDIA GPU Operator on OKD. It is provided as is and is only developed to work
on the my OKD cluster. Feel free to use, adapt this as needed or provide feedback
and improvements. I may integrate feedback and improvements if they fit my usage.

# Usage
In order to use this you need to adapt multiple version numbers at multiple locations
— its just a "works-for-me"!
This includes driver, scos and kernel versions.
Once all versions match your requirements everything can be applied to an OKD cluster
and builds can be started. It is important to note here that the actual driver
container build depends on a successful build of the kernel and cuda images.
This kernel image provides kernel headers that are otherwise unavailable, see okd-project/okd#2061
for details. The script used to generate kernel header packages automatically detect
the current kernel version and may need to be adapted if the build should be
done for other versions (before an update is performed for example).

> A unified setting for all versions at one location would be nice as an enhancement ;)

An build of all images may be triggered using these commands:
```
oc -n nvidia-gpu-operator start-build cuda && \
oc -n nvidia-gpu-operator start-build kernel -w && \
oc -n nvidia-gpu-operator start-build driver-pb
```

In order to deploy this driver container the following Helm Parameters are used:
```yaml
  - chart: gpu-operator
    repoURL: https://helm.ngc.nvidia.com/nvidia
    targetRevision: v25.3.0
    helm:
      releaseName: gpu-operator
      parameters:
        - name: "nfd.enabled"
          value: "false"
        - name: "enable_selinux"
          value: "true"
        - name: "platform.openshift"
          value: "true"
        - name: "operator.use_ocp_driver_toolkit"
          value: "true"
        - name: "driver.repository"
          value: "image-registry.openshift-image-registry.svc:5000/nvidia-gpu-operator"
        - name: "driver.imagePullPolicy"
          value: "Always"
        - name: "driver.usePrecompiled"
          value: "true"
        - name: "driver.version"
          value: "570"
        - name: "dcgmExporter.serviceMonitor.enabled"
          value: "true"
```
Copied out of my ArgoCD application manifest managing the operator.

# Known problems
Depending on your setup it might be necessary to manually load the `video` kernel
module, at least it was the case at my machines. The driver container will try
to do it itself, but as kernel versions in the DTK image do not match it will fail
and need to be done on the host itself (or modules need to be rebuild completely — not implemented).
I use the following machineconfig objects to do so:

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: '0'
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 98-worker-kmod-video
spec:
  config:
    ignition:
      version: 3.4.0
    storage:
      files:
        - contents:
            compression: gzip
            source: 'data:;base64,H4sIAAAAAAAAAyXMSwrAIAxF0bmreOBeuo/UpCUQjVgV3H1/43s5EZu3JDAnxlQWR3YeJqALLFUKS0kLhzeUpyvhrAPcdEoL8b20I9NC8Y79h4RBo3umronMVvjgcAPjPbbmbAAAAA=='
          mode: 420
          overwrite: true
          path: /etc/modules-load.d/video.conf
```
