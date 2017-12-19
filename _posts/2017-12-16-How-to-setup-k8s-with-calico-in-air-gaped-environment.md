---
layout: post
title: "Deploy K8s + Calico network in air-gapped environment"
date: 2017-12-18 14:16:01 +0800
tags: ["Kubenetes","Calico"]
---

## Pre-defined environment

- Internet availabe host: **localhost** , the host have to be with the same architecture as air-gaped host ,like amd_64, x86_64
- Air-gapped host: **master01.airgaped.org**, **node01.airgaped.org**, **node02.airgaped.org**,  **node03.airgaped.org**
- K8s master runs on **master01.airgaped.org**
- Private registry service runs on **registry.airgaped.org** (hosted on **master01.airgaped.org**) with port 443 and TLS enabled. See [Deploy a private docker registry in air-gaped environment](https://highland0971.github.io/2017/12/18/Deploy-a-private-docker-registry-in-air-gaped-environment.html)


The following process is composed under the instruction of [Creating a Custom Cluster from Scratch](https://kubernetes.io/docs/getting-started-guides/scratch/#downloading-and-extracting-kubernetes-binaries) on Kubernetes.io. You may refer to orginal documents to get detailed information.


## 1. Download K8s Packages

 1. As described in [Downloading and Extracting Kubernetes Binaries](https://kubernetes.io/docs/getting-started-guides/scratch/#downloading-and-extracting-kubernetes-binaries), you have to download the latest kubernetes bootstrap binaries from [Github](https://github.com/kubernetes/kubernetes/releases), on **localhost** ：

    ```bash
    curl -L -o kubernetes-v1_9_0.tar.gz \ https://github.com/kubernetes/kubernetes/releases/download/v1.9.0/kubernetes.tar.gz
    tar -zxf kubernetes-v1_9_0.tar.gz 
    pushd kubernetes/cluster
    ./get-kube-binaries.sh
    ```

    You may have to try multiple times as poor Internet connection from CN to dl.k8s.io.

    After successfully download, you can archive download binaries as tarball and copy to air-gapped hosts and place them in proper directory.

    on **localhost** ：

    ```bash
    popd
    tar -czf kubernetes.downloaded.tar.gz kubernetes
    scp kubernetes.downloaded.tar.gz master01.airgpaped.org
    ```

    on **master01.airgapped.org** ：

    ```bash
    tar -zxf kubernetes.downloaded.tar.gz
    sudo mv kubernetes /opt
    ```

 2. Perpare for required docker image



  ```
  {
   "exec-opts": ["native.cgroupdriver=systemd"]
  }
  ```

        4. Disable SeLinux and enable ipv4 forward

  ```
  sudo setenforce 0
  sudo vi /etc/selinux/config
  # set SELINUX=disabled
  ```
  SELINUX must be set disabled, otherwise the docker contianer can not access host path file , encounter problems like level=fatal msg="open /cert/ca.crt: permission denied"

  Enable ipv4 forwarding by sysctl, net.ipv4.ip_forward = 1

        5. Disable Swap file by disable swap sector in `/etc/fstab` and run 
  ```
  sudo swapoff -aV
  ```

    6. Enable&Run docker and kubelet
  ```
  sudo systemctl enable docker && sudo systemctl start docker
  #sudo systemctl enable kubelet && sudo systemctl start kubelet

## Appendix 1. Setup the kvm-qemu virtual machine with virt-install

  ```
 qemu-image create 
 virt-install -n master --memory 3072 --vcpus 2 \
  -l /tmp/CentOS-7-x86_64-Minimal-1708.iso \
  -w network=default --graphics=none \
  --disk path=/mnt/slow_storage/master.qcow2 \
  --os-type=linux \
  --extra-args='console=tty0 console=ttyS0,115200n8 serial'
```

## 
```