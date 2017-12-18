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
  ````
  
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
  ```
  
  7. Generate certificates for local registry
  ```
  openssl genrsa -out ca.key 2048
  openssl genrsa -out local-registry.key 2048
  vi local-registry.conf # replace <*> with acturall name
  
  #----------
  #[ req ]
  #default_bits = 2048
  #prompt = no
  #default_md = sha256
  #req_extensions = req_ext
  #distinguished_name = dn

  #[ dn ]
  #C = cn
  #ST = Tianjin
  #L = Tianjin
  #O = TJMCC
  #OU = JZXN
  #CN = master

  #[ req_ext ]
  #subjectAltName = @alt_names

  #[ alt_names ]
  #DNS.1 = master
  #DNS.2 = master.default
  #DNS.3 = master.default.svc
  #DNS.4 = master.default.svc.cluster
  #DNS.5 = master.default.svc.cluster.local
  #IP.1 = 192.168.122.48
  #IP.2 = 192.168.122.48

  #[ v3_ext ]
  #authorityKeyIdentifier=keyid,issuer:always
  #basicConstraints=CA:FALSE
  #keyUsage=keyEncipherment,dataEncipherment
  #extendedKeyUsage=serverAuth,clientAuth
  #subjectAltName=@alt_names
  
  #----------
  
  openssl req -new -key local-registry.key -out local-registry.csr -config local-registry.conf
  
  openssl x509 -req -in local-registry.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out local-registry.crt -days 10000 \
  -extensions v3_ext -extfile local-registry.conf
  
  openssl x509  -noout -text -in ./local-registry.crt
  
  sudo mkdir /etc/docker/certs.d/master:5000 -p
  sudo cp ca.crt /etc/docker/certs.d/master:5000
  
  sudo mkdir /var/certs
  sudo cp local-registry.* /var/certs
  
  ```
  
  7. Get local registry image from docker.io and start up
  
  This images need firewalld service started up to setup iptable rules
  
  ```
  docker pull registry:latest #On internet available host
  docker save docker.io/registry > docker-io-registry.tar #On internet available host
  scp docker-io-registry.tar user@host:~/ #On internet available host to air-gaped host 
  ssh user@host
  docker load -i ~/docker-io-registry.tar
  sudo mkdir /mnt/docker-images
  sudo chown $USER@docker /mnt/docker-images
  
   docker run -d --restart=always -v /mnt/docker-images:/var/lib/registry \
   -v /var/certs:/cert -e REGISTRY_HTTP_TLS_CERTIFICATE=/cert/local-registry.crt \
   -e REGISTRY_HTTP_TLS_KEY=/cert/local-registry.key \
   -p 5000:5000 --name local-registry registry
   
   docker logs local-registry #check  msg="listening on [::]:5000, tls"
   
  ```
  
  8.Upload docker images needed by kubenates to local-registry
 
  ```
  
  docker load -i 
  docker push master:5000/kube-proxy
  docker rmi master:5000/kube-proxy:v1.9.0
  docker pull master:5000/kube-proxy:v1.9.0
  
  ```
