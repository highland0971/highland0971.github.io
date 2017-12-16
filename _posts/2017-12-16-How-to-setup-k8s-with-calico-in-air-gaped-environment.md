---
layout: post
title: "如何在网络隔离环境下安装K8s+Calico"
date: 2017-12-16 14:16:01 +0800
tags: ["Kubenetes","Calico"]
---

1. Setup the kvm-qemu virtual machine with virt-install
 
 ```
 virt-install -n master --memory 3072 --vcpus 2 \
 
  -l /tmp/CentOS-7-x86_64-Minimal-1708.iso \
  
  -w network=default --graphics=none \
  
  --disk path=/mnt//slow_storage/master.qcow2 \
  
  --os-type=linux \
  
  --extra-args='console=tty0 console=ttyS0,115200n8 serial'
  ```
  
  2. Download Docker RPM and K8s Packages
  
  ```
  cd && mkdir kube-packages
  ```
  
  ### For Docker
  ```
  yum reinstall --downloadonly  --downloaddir=./ docker 
  ```
  
  ### For Kubenetes with kubeadm
  ```
  cat <<EOF > /etc/yum.repos.d/kubernetes.repo
  [kubernetes]
  name=Kubernetes
  baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
  enabled=1
  gpgcheck=1
  repo_gpgcheck=1
  gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
  EOF
  yum reinstall --downloadonly  --downloaddir=./ kubelet kubeadm kubectl
  ```
  ### scp package to air-gaped environment and install
  ```
   ssh user@host mkdir ~/packages
   scp * user@host:~/packages
   ssh user@host
   cd ~/packages
   sudo rpm -ivh *.rpm 
  ```
  
  
