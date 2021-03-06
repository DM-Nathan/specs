```text
SPDX-License-Identifier: Apache-2.0
Copyright (c) 2019 Intel Corporation
```

- [Userspace CNI](#userspace-cni)
  - [Setup Userspace CNI](#setup-userspace-cni)
  - [HugePages configuration](#hugepages-configuration)
  - [Pod deployment](#pod-deployment)
  - [Virtual interface usage](#virtual-interface-usage)

# Userspace CNI

Userspace CNI is a Container Network Interface Kubernetes plugin that was designed to simplify the process of deployment of DPDK based applications in Kubernetes pods. The plugin uses Kubernetes and Multus CNI's CRD to provide pod with virtual DPDK-enabled ethernet port. In this document you can find details about how to install OpenNESS with Userspace CNI support and how to use it's main features.

## Setup Userspace CNI

OpenNESS for Network Edge has been integrated with Userspace CNI to allow user to easily run DPDK based applications inside Kubernetes pods. To install OpenNESS Network Edge with Userspace CNI support, please add value `userspace` to variable `kubernetes_cnis` in `group_vars/all.yml` and set value of the variable `ovs_dpdk` in `roles/kubernetes/cni/kubeovn/common/defaults/main.yml` to `true`:

```yaml
# group_vars/all.yml
kubernetes_cnis:
- kubeovn
- userspace
```

```yaml
# roles/kubernetes/cni/kubeovn/common/defaults/main.yml
ovs_dpdk: true
```

## HugePages configuration

Please be aware that DPDK apps will require specific amount of HugePages enabled. By default the ansible scripts will enable 1024 of 2M HugePages in system, and then start OVS-DPDK with 1Gb of those HugePages. If you would like to change this settings to reflect your specific requirements please set ansible variables as defined in the example below. This example enables 4 of 1GB HugePages and appends 1 GB to OVS-DPDK leaving 3 pages for DPDK applications that will be running in the pods.

```yaml
# network_edge.yml
- hosts: controller_group
  vars:
    hugepage_amount: "4"

- hosts: edgenode_group
  vars:
    hugepage_amount: "4"
```

```yaml
# roles/machine_setup/grub/defaults/main.yml
hugepage_size: "1G"
```

>The variable `hugepage_amount` that can be found in `roles/machine_setup/grub/defaults/main.yml` can be left at default value of `5000` as this value will be overridden by values of `hugepage_amount` variables that were set earlier in `network_edge.yml`.

```yaml
# roles/kubernetes/cni/kubeovn/common/defaults/main.yml
ovs_dpdk_hugepage_size: "1Gi" # This is the size of single hugepage to be used by DPDK. Can be 1Gi or 2Mi.
ovs_dpdk_hugepages: "1Gi" # This is overall amount of hugepags available to DPDK.
```

## Pod deployment

To deploy pod with DPDK interface please create pod with `hugepages` mounted to `/dev/hugepages`, host directory `/var/run/openvswitch/` (with mandatory trailing slash character) mounted into pod with the volume name `shared-dir` (the name `shared-dir` is mandatory) and `userspace-openness` network annotation. You can find example pod definition with two DPDK ports below:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: userspace-example
  annotations:
    k8s.v1.cni.cncf.io/networks: userspace-openness, userspace-openness
spec:
  containers:
  - name: userspace-example
    image: image-name
    imagePullPolicy: Never
    securityContext:
      privileged: true
    volumeMounts:
    - mountPath: /ovs
      name: shared-dir
    - mountPath: /dev/hugepages
      name: hugepages
    resources:
      requests:
        memory: 1Gi
      limits:
        hugepages-1Gi: 2Gi
    command: ["sleep", "infinity"]
  volumes:
  - name: shared-dir
    hostPath:
      path: /var/run/openvswitch/
  - name: hugepages
    emptyDir:
      medium: HugePages
```

## Virtual interface usage

Socket files for virtual interfaces generated by Userspace CNI are created on host machine in `/var/run/openvswitch` directory. This directory has to be mounted into your pod by volume with **obligatory name `shared-dir`** (in our [example pod definition](#pod-deployment) `/var/run/openvswitch` is mounted to pod as `/ovs`). Then you can use sockets available in your mount-point directory in your DPDK-enabled application deployed inside pod. You can find further example in [Userspace CNI's documentation](https://github.com/intel/userspace-cni-network-plugin#testing-with-dpdk-testpmd-application).
