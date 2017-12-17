---
layout: post
title: "如何在网络隔离环境下安装K8s+Calico"
date: 2017-12-16 14:16:01 +0800
tags: ["Kubenetes","Calico"]
---

## 1. Setup the kvm-qemu virtual machine with virt-install
 
 ```
 virt-install -n master --memory 3072 --vcpus 2 \
  -l /tmp/CentOS-7-x86_64-Minimal-1708.iso \
  -w network=default --graphics=none \
  --disk path=/mnt//slow_storage/master.qcow2 \
  --os-type=linux \
  --extra-args='console=tty0 console=ttyS0,115200n8 serial'
  ```
  
 ## 2. Download & install Docker RPM and K8s Packages
  
  ```
  cd && mkdir kube-packages
  
  ### For Docker
  
  ```
  yum reinstall --downloadonly  --downloaddir=./ docker 
  ssh user@host mkdir ~/packages
  scp * user@host:~/packages
  ssh user@host
  cd ~/packages
  sudo rpm -ivh *.rpm 
  sudo groupadd docker
  sudo usermod -aG docker $USER
  sudo reboot
  ```
  
  3. Configure docker cgroup driver compalince with k8s, if is not specified in the systemd ExecStart cmd opts. Add fowllowing into /etc/docker/daemon.json
  
  ```
  {
   "exec-opts": ["native.cgroupdriver=systemd"]
  }
  ```
  
  4. Disable SeLinux and enable ipv4 forward
  
  ```
  sudo setenforce 0
  sudo vi /etc/selinux/config
  SELINUX=disabled
  ```
  Enable ipv4 forwarding sysctl, net.ipv4.ip_forward = 1
  
  
  5. Disable Swap file by disable swap sector in `/etc/fstab` and run 
  ```
  sudo swapoff -aV
  ```
  6. Enable&Run docker and kubelet
  ```
  sudo systemctl enable docker && sudo systemctl start docker
  sudo systemctl enable kubelet && sudo systemctl start kubelet
  ```
  
  7. Get local registry image from docker.io and start up
  
  This images need firewalld service started up to setup iptable rules
  
  ```
  docker pull registry:latest
  docker save docker.io/registry > docker-io-registry.tar
  scp docker-io-registry.tar user@host:~/
  ssh user@host
  docker load -i ~/docker-io-registry.tar
  sudo mkdir /mnt/docker-images
  sudo chown $USER@docker /mnt/docker-images
  docker run -d --restart=always \
  -v /mnt/docker-images:/var/lib/registry \
  -v /cert:/cert --name local-registry registry \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/cert/ca.crt \
  -e REGISTRY_HTTP_TLS_KEY=/cert/ca.key \
  -p 5000:443
  ```
