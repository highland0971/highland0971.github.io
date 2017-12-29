---
layout: post
title: "Deploy K8s + Calico Network in air-gapped environment within 20 minutes"
date: 2017-12-29 14:16:01 +0800
tags: ["Kubenetes","Calico"]
---
This is a verified installation documents for `K8S 1.9.0` + `Calico 2.6.5` + `Docker 1.12.6` + `etcd 3.2.12` + `kube-dns 1.14.7`.

In this article we focus on how to setup a complete functional [Kubernetes](https://kubernetes.io/) cluster with [Calico](https://docs.projectcalico.org/v2.6/introduction/) Network solution and [Docker Private Registry](https://docs.docker.com/registry/deploying/) in an air-gapped environment, which is the common scene for lots of companies.

## 1. Pre-defined environment vars
- The following pre-defined env-vars should set on the `Internet available host` (for packages and binaries pre-download) and `INSTALLATION_BASE` host (Which can access air-gapped deployment cluster, for installation launch.)

```bash
export DOCKER_CLIENT_USER=highland #For users which need docker CLI access
export K8S_NODE01=jin #Kubernetes target deploy node
export K8S_NODE02=mu
export K8S_NODE03=shui
export K8S_NODE04=huo
export K8S_NODE01_IP=192.168.122.180 #Kubernetes target deploy node ip
export K8S_NODE02_IP=192.168.122.117
export K8S_NODE03_IP=192.168.122.52
export K8S_NODE04_IP=192.168.122.102
export K8S_NODES="$K8S_NODE01 $K8S_NODE02 $K8S_NODE03 $K8S_NODE04"
export ETCD_CLIENT_PORT=2379 #ETCD daemon client listen port
export ETCD_PEER_PORT=2380  #ETCD daemon peer listen port
export ETCD1_IP=$K8S_NODE02_IP  #ETCD daemon node ip
export ETCD2_IP=$K8S_NODE03_IP
export ETCD3_IP=$K8S_NODE04_IP
export ETCD_IP_ARRAY=($ETCD1_IP $ETCD2_IP $ETCD3_IP)
export ETCD_NODES="$K8S_NODE02 $K8S_NODE03 $K8S_NODE04"
export ETCD_NODE_ARRAY=($K8S_NODE02 $K8S_NODE03 $K8S_NODE04)
export ETCD_ENDPOINTS=https://$ETCD1_IP:$ETCD_CLIENT_PORT,https://$ETCD2_IP:$ETCD_CLIENT_PORT,https://$ETCD3_IP:$ETCD_CLIENT_PORT
export PRIVATE_REGISTRY_SERV=$K8S_NODE02 #Private docker registry deployment node
export PRIVATE_REGISTRY_SERV_IP=$K8S_NODE02_IP
export PRIVATE_REGISTRY=${PRIVATE_REGISTRY_SERV}:443 #Private docker registry service address
export KUBE_MASTER_SERV_01=$K8S_NODE01 #The Non-HA K8S cluster master node
export KUBE_MASTER_SERV_IP_01=$K8S_NODE01_IP
export KUBE_APISERVER_IP_01=$KUBE_MASTER_SERV_IP_01 #The Non-HA K8S cluster apiserver node
export KUBE_APISERVER_PORT=6443
export KUBE_APISERVER=https://$KUBE_APISERVER_IP_01:${KUBE_APISERVER_PORT}
export KUBE_CLUSTER_SERVICE_IP=172.16.0.1  #The kubernetes cluster ip address
export KUBE_CLUSTER_SERVICE_CIDR=172.16.0.1/16  #The kubernetes cluster CIDR address
export KUBE_CLUSTER_POD_CIDR=172.17.0.1/16  #The kubernetes cluster pods CIDR address
export CLUSTER_DNS_IP=172.16.0.10 #The kubernetes kube-dns service address

export K8S_BASE_VER=v1.9.0
export K8S_DNS_VER=1.14.7
export K8S_PAUSE_VER=3.1
export CALICO_CONTROLLER_VER=v1.0.2
export CALICO_NODE_VER=v2.6.5
export CALICO_CNI_VER=v1.11.2

export K8S_PROXY_VER=${K8S_BASE_VER}
export K8S_PAUSE_IMG_NAME=pause-amd64
export K8S_PROXY_IMG_NAME=kube-proxy
export K8S_KUBE_DNS_IMG_NAME=k8s-dns-kube-dns-amd64
export K8S_DNSMASQ_IMG_NAME=k8s-dns-dnsmasq-nanny-amd64
export K8S_SIDECAR_IMG_NAME=k8s-dns-sidecar-amd64

export CALICO_CONTROLLER_IMG_NAME=kube-controllers
export CALICO_NODE_IMAGE_NAME=node
export CALICO_CNI_IMAGE_NAME=cni


export GCR_IMAGES="kube-aggregator kube-controller-manager ${K8S_PROXY_IMG_NAME} kube-scheduler kube-apiserver"
```

- Run this on installation base host(**INSTALLATION_BASE**)

```bash
#Change the hostname on each node if needed
for node in $K8S_NODES ; do
  ssh $node hostnamectl set-hostname $node
done

#Setup ssh passwordless login and hostname resolve
for node in $K8S_NODES ; do
  ssh-copy-id -f $node
  ssh $node "echo $K8S_NODE01_IP $K8S_NODE01 >> /etc/hosts"
  ssh $node "echo $K8S_NODE02_IP $K8S_NODE02 >> /etc/hosts"
  ssh $node "echo $K8S_NODE03_IP $K8S_NODE03 >> /etc/hosts"
  ssh $node "echo $K8S_NODE04_IP $K8S_NODE04 >> /etc/hosts"
done
```

## 2. Download required images and binaries for **K8S**,**Etcd**,**Docker**,**Calico**,**cfssl**
- Run the following scripts on Internet available hosts with `Docker` pre-installed

```bash
cd
mkdir -p all-in-one/bin
mkdir -p all-in-one/images
mkdir -p all-in-one/rpms
mkdir -p all-in-one/mixture
mkdir -p all-in-one/configs

# For Docker,recommend run the following command on a clean linux host without and packages post installed
yum update && yum install -y --downloadonly  --downloaddir=all-in-one/rpms docker
yum reinstall -y --downloadonly --downloaddir=all-in-one/rpms docker

#For kubernetes client & server binary
curl -L https://dl.k8s.io/${K8S_BASE_VER}/kubernetes-client-linux-amd64.tar.gz -o all-in-one/mixture/kubernetes-client-linux-amd64.tar.gz
curl -L https://dl.k8s.io/${K8S_BASE_VER}/kubernetes-server-linux-amd64.tar.gz -o all-in-one/mixture/kubernetes-server-linux-amd64.tar.gz

# For kube-dns
for img in $K8S_KUBE_DNS_IMG_NAME $K8S_DNSMASQ_IMG_NAME $K8S_SIDECAR_IMG_NAME ; do
  echo tar highland0971/${img}:${K8S_DNS_VER}
  docker pull highland0971/${img}:${K8S_DNS_VER} && \
  docker tag highland0971/${img}:${K8S_DNS_VER} ${PRIVATE_REGISTRY}/${img}:${K8S_DNS_VER} && \
  docker save ${PRIVATE_REGISTRY}/${img}:${K8S_DNS_VER} -o all-in-one/images/${img}:${K8S_DNS_VER}.tar
done

docker pull highland0971/${K8S_KUBE_DNS_IMG_NAME}:${K8S_DNS_VER} && \
docker tag highland0971/${K8S_KUBE_DNS_IMG_NAME}:${K8S_DNS_VER} ${PRIVATE_REGISTRY}/${K8S_KUBE_DNS_IMG_NAME}:${K8S_DNS_VER} && \
docker save ${PRIVATE_REGISTRY}/${K8S_KUBE_DNS_IMG_NAME}:${K8S_DNS_VER} -o all-in-one/images/${K8S_KUBE_DNS_IMG_NAME}:${K8S_DNS_VER}.tar

# For pause
docker pull highland0971/${K8S_PAUSE_IMG_NAME}:${K8S_PAUSE_VER} && \
docker tag highland0971/${K8S_PAUSE_IMG_NAME}:${K8S_PAUSE_VER} ${PRIVATE_REGISTRY}/${K8S_PAUSE_IMG_NAME}:${K8S_PAUSE_VER} && \
docker save ${PRIVATE_REGISTRY}/${K8S_PAUSE_IMG_NAME}:${K8S_PAUSE_VER} -o all-in-one/images/${K8S_PAUSE_IMG_NAME}:${K8S_PAUSE_VER}.tar

# For registry
docker pull registry && \
docker save docker.io/registry:latest -o all-in-one/images/docker-io.registry.tar

# For cfssl tools
export CFSSL_URL="https://pkg.cfssl.org/R1.2"
curl -L "${CFSSL_URL}/cfssl_linux-amd64" -o all-in-one/bin/cfssl
curl -L "${CFSSL_URL}/cfssljson_linux-amd64" -o all-in-one/bin/cfssljson
chmod +x all-in-one/bin/cfssl*

# For etcd
export ETCD_URL="https://github.com/coreos/etcd/releases/download"
curl -L "${ETCD_URL}/v3.2.12/etcd-v3.2.12-linux-amd64.tar.gz" -o all-in-one/bin/etcd-v3.2.12-linux-amd64.tar.gz

# For Calico
export CALICO_URL="https://docs.projectcalico.org/v2.6/getting-started/kubernetes/installation"
curl -L "${CALICO_URL}/rbac.yaml" -o all-in-one/configs/calico-rbac.yaml
curl -L "${CALICO_URL}/hosted/calico.yaml" -o all-in-one/configs/calico-deploy.yaml.conf

docker pull quay.io/calico/${CALICO_CONTROLLER_IMG_NAME}:${CALICO_CONTROLLER_VER} && \
docker tag quay.io/calico/${CALICO_CONTROLLER_IMG_NAME}:${CALICO_CONTROLLER_VER} ${PRIVATE_REGISTRY}/calico/${CALICO_CONTROLLER_IMG_NAME}:${CALICO_CONTROLLER_VER} && \
docker save ${PRIVATE_REGISTRY}/calico/${CALICO_CONTROLLER_IMG_NAME}:${CALICO_CONTROLLER_VER} -o all-in-one/images/calico_${CALICO_CONTROLLER_IMG_NAME}:${CALICO_CONTROLLER_VER}.tar

docker pull quay.io/calico/${CALICO_NODE_IMAGE_NAME}:${CALICO_NODE_VER} && \
docker tag quay.io/calico/${CALICO_NODE_IMAGE_NAME}:${CALICO_NODE_VER} ${PRIVATE_REGISTRY}/calico/${CALICO_NODE_IMAGE_NAME}:${CALICO_NODE_VER} && \
docker save ${PRIVATE_REGISTRY}/calico/${CALICO_NODE_IMAGE_NAME}:${CALICO_NODE_VER} -o all-in-one/images/calico_${CALICO_NODE_IMAGE_NAME}:${CALICO_NODE_VER}.tar

docker pull quay.io/calico/${CALICO_CNI_IMAGE_NAME}:${CALICO_CNI_VER} && \
docker tag quay.io/calico/${CALICO_CNI_IMAGE_NAME}:${CALICO_CNI_VER} ${PRIVATE_REGISTRY}/calico/${CALICO_CNI_IMAGE_NAME}:${CALICO_CNI_VER} && \
docker save ${PRIVATE_REGISTRY}/calico/${CALICO_CNI_IMAGE_NAME}:${CALICO_CNI_VER} -o all-in-one/images/calico_${CALICO_CNI_IMAGE_NAME}:${CALICO_CNI_VER}.tar

# Archive and check
tar -czf all-in-one.tar.gz all-in-one
tar -ztf all-in-one.tar.gz
# all-in-one/
# all-in-one/bin/
# all-in-one/bin/cfssl
# all-in-one/bin/cfssljson
# all-in-one/bin/etcd-v3.2.12-linux-amd64.tar.gz
# all-in-one/images/
# all-in-one/images/pause-amd64:3.1.tar
# all-in-one/images/docker-io.registry.tar
# all-in-one/images/k8s-dns-kube-dns-amd64:1.14.7.tar
# all-in-one/images/calico_kube-controllers:v1.0.2.tar
# all-in-one/images/calico_node:v2.6.5.tar
# all-in-one/images/calico_cni:v1.11.2.tar
# all-in-one/images/k8s-dns-dnsmasq-nanny-amd64:1.14.7.tar
# all-in-one/images/k8s-dns-sidecar-amd64:1.14.7.tar
# all-in-one/rpms/
# all-in-one/rpms/docker-1.12.6-68.gitec8512b.el7.centos.x86_64.rpm
# all-in-one/rpms/audit-libs-python-2.7.6-3.el7.x86_64.rpm
# all-in-one/rpms/checkpolicy-2.5-4.el7.x86_64.rpm
# all-in-one/rpms/container-selinux-2.33-1.git86f33cd.el7.noarch.rpm
# all-in-one/rpms/container-storage-setup-0.8.0-3.git1d27ecf.el7.noarch.rpm
# all-in-one/rpms/docker-client-1.12.6-68.gitec8512b.el7.centos.x86_64.rpm
# all-in-one/rpms/docker-common-1.12.6-68.gitec8512b.el7.centos.x86_64.rpm
# all-in-one/rpms/libcgroup-0.41-13.el7.x86_64.rpm
# all-in-one/rpms/libsemanage-python-2.5-8.el7.x86_64.rpm
# all-in-one/rpms/oci-register-machine-0-3.13.gitcd1e331.el7.x86_64.rpm
# all-in-one/rpms/oci-systemd-hook-0.1.14-1.git1ba44c6.el7.x86_64.rpm
# all-in-one/rpms/oci-umount-2.3.0-1.git51e7c50.el7.x86_64.rpm
# all-in-one/rpms/policycoreutils-python-2.5-17.1.el7.x86_64.rpm
# all-in-one/rpms/python-IPy-0.75-6.el7.noarch.rpm
# all-in-one/rpms/setools-libs-3.3.8-1.1.el7.x86_64.rpm
# all-in-one/rpms/skopeo-containers-0.1.26-2.dev.git2e8377a.el7.centos.x86_64.rpm
# all-in-one/rpms/yajl-2.0.4-4.el7.x86_64.rpm
# all-in-one/mixture/
# all-in-one/mixture/kubernetes-server-linux-amd64.tar.gz
# all-in-one/mixture/kubernetes-client-linux-amd64.tar.gz
# all-in-one/configs/
# all-in-one/configs/calico-rbac.yaml
# all-in-one/configs/calico-deploy.yaml
# all-in-one/configs/calico-deploy.yaml.conf
```

- Distribute to target installation base host
You can change the distribution method depends on your environment

```bash
# define ${INSTALLATION_BASE} as you wanted
scp all-in-one.tar.gz ${INSTALLATION_BASE}:~/
ssh ${INSTALLATION_BASE} tar -zxf ~/all-in-one.tar.gz
```

# Launch the following scripts on **INSTALLATION_BASE** host

## 3. Prepare kubelet runtime environment

```bash
for node in ${K8S_NODES} ;do
  echo Disable swap file on ${node}
  ssh ${node} swapoff -a
  ssh ${node} "grep 'swap' /etc/fstab && cp /etc/fstab /etc/fstab.with_swap && echo orginal /etc/fstab file has been backup to etc/fstab.with_swap "
  ssh ${node} sed -i '/swap/d' /etc/fstab

  echo Disable SeLinux on ${node}
  ssh $node setenforce 0
  ssh $node "sed -i 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/selinux/config"

  echo Disable firewalld on ${node}
  ssh $node systemctl stop firewalld
  ssh $node systemctl disable firewalld
done
```

## 4. Install `Docker`
Install `Docker` on target hosts

```bash
# Scp docker rpm packages to each nodes, passwdless environment must be set previously for root user
# Execute current command as root

for node in ${K8S_NODES} ; do
  ssh $node mkdir ~/rpms -p
  scp all-in-one/rpms/* ${node}:~/rpms
  ssh $node rpm -ivh ~/rpms/*.rpm
  ssh $node groupadd docker
  ssh $node usermod -aG docker $DOCKER_CLIENT_USER
  ssh $node "sed -i '/^ExecReload/i\ExecStartPost=\/sbin\/iptables -I FORWARD -s 0.0.0.0\/0 -j ACCEPT' /lib/systemd/system/docker.service"
  ssh $node systemctl disable docker
  ssh $node systemctl enable docker
  ssh $node systemctl stop docker
  ssh $node systemctl start docker
done
```
## 5. Setup private registry service

```bash
cd
cp all-in-one/bin/cfssl* /usr/local/bin/
scp all-in-one/images/docker-io.registry.tar ${PRIVATE_REGISTRY_SERV}:~/
ssh ${PRIVATE_REGISTRY_SERV} docker load -i ~/docker-io.registry.tar
ssh ${PRIVATE_REGISTRY_SERV} mkdir /etc/docker/ssl -p

mkdir certs/docker -p
#Generate docker CA
cat <<EOF > certs/ca-config.json
{
  "signing":{
    "default":{
      "expiry":"87600h"
    },
    "profiles":{
      "default":{
        "usages":[
        "signing",
        "key encipherment",
        "server auth",
        "client auth"
        ],
        "expiry":"87600h"
      }
    }
  }
}
EOF

cat <<EOF > certs/docker/docker-ca-csr.json
{
  "CN":"docker-ca",
  "key":{
    "algo":"rsa",
    "size":2048
  },
  "names":[
    {
      "C":"cn",
      "ST":"DreamState",
      "L":"DreamCity",
      "O":"DreamOrganization",
      "OU":"DockerPrivateRegistry"
    }
  ]
}
EOF

cfssl gencert -initca certs/docker/docker-ca-csr.json | cfssljson -bare certs/docker/docker-ca

#Generate docker TLS certificates
cat <<EOF > certs/docker/private-registry-csr.json
{
  "CN":"private-registry",
  "hosts":[
    "127.0.0.1",
    "${PRIVATE_REGISTRY_SERV_IP}",
    "${PRIVATE_REGISTRY_SERV}"    
  ],
  "key":{
    "algo":"rsa",
    "size":2048
  },
  "names":[
    {
      "C":"cn",
      "ST":"DreamState",
      "L":"DreamCity",
      "O":"DreamOrganization",
      "OU":"DockerPrivateRegistry"
    }
  ]
}
EOF

cfssl gencert -ca=certs/docker/docker-ca.pem \
  -ca-key=certs/docker/docker-ca-key.pem \
  -config=certs/ca-config.json \
  -profile=default \
  certs/docker/private-registry-csr.json | cfssljson -bare certs/docker/private-registry

#Make private registry trusted in the nodes by copy Docker CA file to /etc/docker/certs.d/${PRIVATE_REGISTRY}
for node in ${K8S_NODES} ; do
  ssh ${node} mkdir -p /etc/docker/certs.d/${PRIVATE_REGISTRY}
  scp certs/docker/docker-ca.pem ${node}:/etc/docker/certs.d/${PRIVATE_REGISTRY}/ca.crt
done

#Start the private registry service
ssh ${PRIVATE_REGISTRY_SERV} mkdir -p /etc/docker/ssl
scp certs/docker/private-registry*.pem ${PRIVATE_REGISTRY_SERV}:/etc/docker/ssl
ssh ${PRIVATE_REGISTRY_SERV} mkdir -p /mnt/registry
ssh ${PRIVATE_REGISTRY_SERV} "docker run -d --restart=always -v /mnt/registry:/var/lib/registry -v /etc/docker/ssl:/certs -e REGISTRY_HTTP_ADDR=0.0.0.0:443 -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/private-registry.pem -e REGISTRY_HTTP_TLS_KEY=/certs/private-registry-key.pem -p 443:443 --name private-registry registry"
ssh ${PRIVATE_REGISTRY_SERV} docker logs private-registry
```

## 6. Distribute K8s binaries and images to nodes

```bash
cd

tar -zxf all-in-one/mixture/kubernetes-client-linux-amd64.tar.gz
tar -zxf all-in-one/mixture/kubernetes-server-linux-amd64.tar.gz

for node in ${K8S_NODES} ; do
  echo Process node ${node}.....
  scp kubernetes/server/bin/kubectl ${node}:/usr/local/bin
  scp kubernetes/server/bin/kubelet ${node}:/usr/local/bin
  ssh ${node} chmod +x /usr/local/bin/kube*
done

ssh ${PRIVATE_REGISTRY_SERV} mkdir ~/images
scp kubernetes/server/bin/*.tar ${PRIVATE_REGISTRY_SERV}:~/images
scp all-in-one/images/*.tar ${PRIVATE_REGISTRY_SERV}:~/images

ssh ${PRIVATE_REGISTRY_SERV} docker load -i images/${K8S_PAUSE_IMG_NAME}:${K8S_PAUSE_VER}.tar
ssh ${PRIVATE_REGISTRY_SERV} docker push  ${PRIVATE_REGISTRY}/${K8S_PAUSE_IMG_NAME}:${K8S_PAUSE_VER}
ssh ${PRIVATE_REGISTRY_SERV} docker rmi  ${PRIVATE_REGISTRY}/${K8S_PAUSE_IMG_NAME}:${K8S_PAUSE_VER}

for img in $K8S_KUBE_DNS_IMG_NAME $K8S_DNSMASQ_IMG_NAME $K8S_SIDECAR_IMG_NAME ; do
  echo push ${PRIVATE_REGISTRY}/${img}:${K8S_DNS_VER}
  ssh ${PRIVATE_REGISTRY_SERV} docker load -i images/${img}:${K8S_DNS_VER}.tar
  ssh ${PRIVATE_REGISTRY_SERV} docker push  ${PRIVATE_REGISTRY}/${img}:${K8S_DNS_VER}
  ssh ${PRIVATE_REGISTRY_SERV} docker rmi  ${PRIVATE_REGISTRY}/${img}:${K8S_DNS_VER}
done

ssh ${PRIVATE_REGISTRY_SERV} docker load -i images/calico_${CALICO_CONTROLLER_IMG_NAME}:${CALICO_CONTROLLER_VER}.tar
ssh ${PRIVATE_REGISTRY_SERV} docker push  ${PRIVATE_REGISTRY}/calico/${CALICO_CONTROLLER_IMG_NAME}:${CALICO_CONTROLLER_VER}
ssh ${PRIVATE_REGISTRY_SERV} docker rmi  ${PRIVATE_REGISTRY}/calico/${CALICO_CONTROLLER_IMG_NAME}:${CALICO_CONTROLLER_VER}

ssh ${PRIVATE_REGISTRY_SERV} docker load -i images/calico_${CALICO_NODE_IMAGE_NAME}:${CALICO_NODE_VER}.tar
ssh ${PRIVATE_REGISTRY_SERV} docker push  ${PRIVATE_REGISTRY}/calico/${CALICO_NODE_IMAGE_NAME}:${CALICO_NODE_VER}
ssh ${PRIVATE_REGISTRY_SERV} docker rmi  ${PRIVATE_REGISTRY}/calico/${CALICO_NODE_IMAGE_NAME}:${CALICO_NODE_VER}

ssh ${PRIVATE_REGISTRY_SERV} docker load -i images/calico_${CALICO_CNI_IMAGE_NAME}:${CALICO_CNI_VER}.tar
ssh ${PRIVATE_REGISTRY_SERV} docker push  ${PRIVATE_REGISTRY}/calico/${CALICO_CNI_IMAGE_NAME}:${CALICO_CNI_VER}
ssh ${PRIVATE_REGISTRY_SERV} docker rmi  ${PRIVATE_REGISTRY}/calico/${CALICO_CNI_IMAGE_NAME}:${CALICO_CNI_VER}

export gcr=gcr.io/google_containers
for img in ${GCR_IMAGES} ; do
  echo push ${PRIVATE_REGISTRY}/${img}:${K8S_BASE_VER}
  ssh ${PRIVATE_REGISTRY_SERV} docker load -i ~/images/${img}.tar
  ssh ${PRIVATE_REGISTRY_SERV} docker tag ${gcr}/${img}:${K8S_BASE_VER}  ${PRIVATE_REGISTRY}/${img}:${K8S_BASE_VER}
  ssh ${PRIVATE_REGISTRY_SERV} docker push  ${PRIVATE_REGISTRY}/${img}:${K8S_BASE_VER}
  ssh ${PRIVATE_REGISTRY_SERV} docker rmi ${gcr}/${img}:${K8S_BASE_VER}
  ssh ${PRIVATE_REGISTRY_SERV} docker rmi  ${PRIVATE_REGISTRY}/${img}:${K8S_BASE_VER}
done

ssh ${PRIVATE_REGISTRY_SERV} rm -rf images
```

## 7. Setup `etcd` daemon
- prepare certificates

```bash
cd
mkdir -p certs/etcd
cat <<EOF > certs/etcd/etcd-ca-csr.json
{
  "CN":"etcd-ca",
  "key":{
    "algo":"rsa",
    "size":2048
  },
  "names":[
    {
      "C":"cn",
      "ST":"DreamState",
      "L":"DreamCity",
      "O":"DreamOrganization",
      "OU":"etcd"
    }
  ]
}
EOF
cfssl gencert -initca certs/etcd/etcd-ca-csr.json | cfssljson -bare certs/etcd/etcd-ca

cat <<EOF > certs/etcd/etcd-csr.json
{
  "CN":"etcd",
  "hosts":[
    "127.0.0.1",
    "${ETCD1_IP}",
    "${ETCD2_IP}",
    "${ETCD3_IP}"
  ],
  "key":{
    "algo":"rsa",
    "size":2048
  },
  "names":[
    {
      "C":"cn",
      "ST":"DreamState",
      "L":"DreamCity",
      "O":"DreamOrganization",
      "OU":"etcd"
    }
  ]
}
EOF

cfssl gencert -ca=certs/etcd/etcd-ca.pem \
  -ca-key=certs/etcd/etcd-ca-key.pem \
  -config=certs/ca-config.json \
  -profile=default \
  certs/etcd/etcd-csr.json | cfssljson -bare certs/etcd/etcd

for node in ${ETCD_NODES} ;do
  echo Copy certs to $node ...
  ssh ${node} mkdir -p /etc/etcd/ssl
  scp certs/etcd/* ${node}:/etc/etcd/ssl
done
```

- prepare binaries and configs

```bash
cd
tar -zxf  all-in-one/bin/etcd-v3.2.12-linux-amd64.tar.gz

let node_idx=0
for node in ${ETCD_NODES} ;do
  echo process node ${node}
  ssh ${node} "systemctl stop etcd.service && systemctl disable etcd.service"
  scp etcd-v3.2.12-linux-amd64/etcd* ${node}:/usr/local/bin
  ssh ${node} groupadd etcd
  ssh ${node} "useradd -c 'Etcd user' -g etcd -s /sbin/nologin -r etcd"
  ssh ${node} mkdir -p /var/lib/etcd
  ssh ${node} chown etcd:etcd -R /var/lib/etcd /etc/etcd
  cat << EOF > etcd.conf
# [member]
ETCD_NAME=${node}
ETCD_DATA_DIR=/var/lib/etcd
ETCD_LISTEN_PEER_URLS=https://0.0.0.0:${ETCD_PEER_PORT}
ETCD_LISTEN_CLIENT_URLS=https://0.0.0.0:${ETCD_CLIENT_PORT}
ETCD_PROXY=off

# [cluster]
ETCD_ADVERTISE_CLIENT_URLS=https://${ETCD_IP_ARRAY[${node_idx}]}:${ETCD_CLIENT_PORT}
ETCD_INITIAL_ADVERTISE_PEER_URLS=https://${ETCD_IP_ARRAY[${node_idx}]}:${ETCD_PEER_PORT}
ETCD_INITIAL_CLUSTER=${ETCD_NODE_ARRAY[0]}=https://${ETCD_IP_ARRAY[0]}:${ETCD_PEER_PORT},${ETCD_NODE_ARRAY[1]}=https://${ETCD_IP_ARRAY[1]}:${ETCD_PEER_PORT},${ETCD_NODE_ARRAY[2]}=https://${ETCD_IP_ARRAY[2]}:${ETCD_PEER_PORT}
ETCD_INITIAL_CLUSTER_STATE=new
ETCD_INITIAL_CLUSTER_TOKEN=etcd-k8s-cluster

# [security]
ETCD_CERT_FILE="/etc/etcd/ssl/etcd.pem"
ETCD_KEY_FILE="/etc/etcd/ssl/etcd-key.pem"
ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/ssl/etcd-ca.pem"
ETCD_AUTO_TLS="true"
ETCD_PEER_CERT_FILE="/etc/etcd/ssl/etcd.pem"
ETCD_PEER_KEY_FILE="/etc/etcd/ssl/etcd-key.pem"
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/ssl/etcd-ca.pem"
ETCD_PEER_AUTO_TLS="true"
EOF

  cat <<EOF > etcd.service
[Unit]
Description=Etcd Service
After=network.target

[Service]
EnvironmentFile=-/etc/etcd/etcd.conf
Type=simple
User=etcd
PermissionsStartOnly=true
ExecStart=/usr/local/bin/etcd
Restart=on-failure
RestartSec=10
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

  ssh ${node} mkdir -p /etc/etcd
  ssh ${node} mkdir -p /var/lib/etcd
  scp etcd.conf ${node}:/etc/etcd
  scp etcd.service ${node}:/lib/systemd/system
  ssh ${node} chown etcd:etcd -R /var/lib/etcd /etc/etcd

  ssh ${node} systemctl enable etcd.service
  ssh ${node} systemctl start etcd.service

  ssh ${node} ETCDCTL_API=3 etcdctl --cacert=/etc/etcd/ssl/etcd-ca.pem --cert=/etc/etcd/ssl/etcd.pem --key=/etc/etcd/ssl/etcd-key.pem  --endpoints="https://${ETCD_IP_ARRAY[${node_idx}]}:${ETCD_CLIENT_PORT}" endpoint health
  ssh ${node} etcdctl --ca-file=/etc/etcd/ssl/etcd-ca.pem -cert-file=/etc/etcd/ssl/etcd.pem -key-file=/etc/etcd/ssl/etcd-key.pem  --endpoints=https://${ETCD_IP_ARRAY[${node_idx}]}:${ETCD_CLIENT_PORT} cluster-health

  let node_idx=${node_idx}+1
done
```

## 8. Prepare certificate for `K8s`

```bash
cd
mkdir -p certs/k8s

#For kubernetes root CA file
cat <<EOF > certs/k8s/k8s-ca-csr.json
{
  "CN":"kubernetes",
  "key":{
    "algo":"rsa",
    "size":2048
  },
  "names":[
    {
      "C":"cn",
      "ST":"DreamState",
      "L":"DreamCity",
      "O":"DreamOrganization",
      "OU":"DreamSystem"
    }
  ]
}
EOF
cfssl gencert -initca certs/k8s/k8s-ca-csr.json | cfssljson -bare certs/k8s/k8s-ca

#For API server certificates
cat <<EOF > certs/k8s/kube-apiserver-csr.json
{
  "CN":"kube-apiserver",
  "hosts":[
    "127.0.0.1",
    "${KUBE_MASTER_SERV_IP_01}",
    "${KUBE_MASTER_SERV_01}",
    "${KUBE_CLUSTER_SERVICE_IP}",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key":{
    "algo":"rsa",
    "size":2048
  },
  "names":[
    {
      "O":"system:masters"
    }
  ]
}
EOF

cfssl gencert -ca=certs/k8s/k8s-ca.pem \
  -ca-key=certs/k8s/k8s-ca-key.pem \
  -config=certs/ca-config.json \
  -profile=default \
  certs/k8s/kube-apiserver-csr.json | cfssljson -bare certs/k8s/kube-apiserver

#For front-proxy-client certificates
cat <<EOF > certs/k8s/kube-front-proxy-client-csr.json
{
  "CN":"front-proxy-client",
  "key":{
    "algo":"rsa",
    "size":2048
  }
}
EOF

cfssl gencert -ca=certs/k8s/k8s-ca.pem \
  -ca-key=certs/k8s/k8s-ca-key.pem \
  -config=certs/ca-config.json \
  -profile=default \
  certs/k8s/kube-front-proxy-client-csr.json | cfssljson -bare certs/k8s/kube-front-proxy-client

#For controller-manager certificates
cat <<EOF > certs/k8s/kube-controller-manager-csr.json
{
  "CN":"system:kube-controller-manager",
  "key":{
    "algo":"rsa",
    "size":2048
  },
  "names":[
    {
      "O":"system:masters"
    }
  ]
}
EOF

cfssl gencert -ca=certs/k8s/k8s-ca.pem \
  -ca-key=certs/k8s/k8s-ca-key.pem \
  -config=certs/ca-config.json \
  -profile=default \
  certs/k8s/kube-controller-manager-csr.json | cfssljson -bare certs/k8s/kube-controller-manager

#For scheduler certificates
cat <<EOF > certs/k8s/kube-scheduler-csr.json
{
  "CN":"system:kube-scheduler",
  "key":{
    "algo":"rsa",
    "size":2048
  },
  "names":[
    {
      "O":"system:masters"
    }
  ]
}
EOF

cfssl gencert -ca=certs/k8s/k8s-ca.pem \
  -ca-key=certs/k8s/k8s-ca-key.pem \
  -config=certs/ca-config.json \
  -profile=default \
  certs/k8s/kube-scheduler-csr.json | cfssljson -bare certs/k8s/kube-scheduler

#For kubelet master certificates
cat <<EOF > certs/k8s/kubelet-master-csr.json
{
  "CN":"system:node:${KUBE_MASTER_SERV_01}",
  "key":{
    "algo":"rsa",
    "size":2048
  },
  "names":[
    {
      "O":"system:nodes"
    }
  ]
}
EOF

cfssl gencert -ca=certs/k8s/k8s-ca.pem \
  -ca-key=certs/k8s/k8s-ca-key.pem \
  -config=certs/ca-config.json \
  -profile=default \
  certs/k8s/kubelet-master-csr.json | cfssljson -bare certs/k8s/kubelet-master

#For admin certificate
cat <<EOF > certs/k8s/kube-admin-csr.json
{
  "CN":"kube-admin",
  "key":{
    "algo":"rsa",
    "size":2048
  },
  "names":[
    {
      "C":"cn",
      "ST":"DreamState",
      "L":"DreamCity",
      "O":"system:masters",
      "OU":"DreamTeam"
    }
  ]
}
EOF

cfssl gencert -ca=certs/k8s/k8s-ca.pem \
  -ca-key=certs/k8s/k8s-ca-key.pem \
  -config=certs/ca-config.json \
  -profile=default \
  certs/k8s/kube-admin-csr.json | cfssljson -bare certs/k8s/kube-admin

#For service account key
openssl genrsa -out certs/k8s/sa.key 2048
openssl rsa -in certs/k8s/sa.key -pubout -out certs/k8s/sa.pub

#For etcd client
cat <<EOF > certs/k8s/kubelet-etcd-client-csr.json
{
  "CN":"kubelet-nodes",
  "key":{
    "algo":"rsa",
    "size":2048
  }
}
EOF

cfssl gencert -ca=certs/etcd/etcd-ca.pem \
  -ca-key=certs/etcd/etcd-ca-key.pem \
  -config=certs/ca-config.json \
  -profile=default \
  certs/k8s/kubelet-etcd-client-csr.json | cfssljson -bare certs/k8s/kubelet-etcd-client

```

## 9. Prepare kubeconfig file for K8s cluster
- Prepare kubeconfig file for each component

```bash
cd
# For admin
kubernetes/server/bin/kubectl config set-cluster kubernetes \
--certificate-authority=certs/k8s/k8s-ca.pem \
--embed-certs=true \
--server=${KUBE_APISERVER} \
--kubeconfig=certs/k8s/admin.conf

kubernetes/server/bin/kubectl config set-credentials kubernetes-admin \
--client-certificate=certs/k8s/kube-admin.pem \
--client-key=certs/k8s/kube-admin-key.pem \
--embed-certs=true \
--kubeconfig=certs/k8s/admin.conf

kubernetes/server/bin/kubectl config set-context default \
--cluster=kubernetes \
--user=kubernetes-admin \
--kubeconfig=certs/k8s/admin.conf

kubernetes/server/bin/kubectl config use-context default \
--kubeconfig=certs/k8s/admin.conf

# For bootstrap
export BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')
cat <<EOF > certs/k8s/token.csv
${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF

kubernetes/server/bin/kubectl config set-cluster kubernetes \
--certificate-authority=certs/k8s/k8s-ca.pem \
--embed-certs=true \
--server=${KUBE_APISERVER} \
--kubeconfig=certs/k8s/bootstrap.conf

kubernetes/server/bin/kubectl config set-credentials kubelet-bootstrap \
--token=${BOOTSTRAP_TOKEN} \
--kubeconfig=certs/k8s/bootstrap.conf

kubernetes/server/bin/kubectl  config set-context default \
--cluster=kubernetes \
--user=kubelet-bootstrap \
--kubeconfig=certs/k8s/bootstrap.conf

kubernetes/server/bin/kubectl config use-context default \
--kubeconfig=certs/k8s/bootstrap.conf

# For kubelet master
kubernetes/server/bin/kubectl config set-cluster kubernetes \
--certificate-authority=certs/k8s/k8s-ca.pem \
--embed-certs=true \
--server=${KUBE_APISERVER} \
--kubeconfig=certs/k8s/kubelet-master.conf

kubernetes/server/bin/kubectl config set-credentials system:node:${KUBE_MASTER_SERV_01} \
--client-certificate=certs/k8s/kubelet-master.pem \
--client-key=certs/k8s/kubelet-master-key.pem \
--embed-certs=true \
--kubeconfig=certs/k8s/kubelet-master.conf

kubernetes/server/bin/kubectl config set-context default \
--cluster=kubernetes \
--user=system:node:${KUBE_MASTER_SERV_01} \
--kubeconfig=certs/k8s/kubelet-master.conf

kubernetes/server/bin/kubectl config use-context default \
--kubeconfig=certs/k8s/kubelet-master.conf

#For controller-manager
kubernetes/server/bin/kubectl config set-cluster kubernetes \
--certificate-authority=certs/k8s/k8s-ca.pem \
--embed-certs=true \
--server=${KUBE_APISERVER} \
--kubeconfig=certs/k8s/controller-manager.conf

kubernetes/server/bin/kubectl config set-credentials system:kube-controller-manager \
--client-certificate=certs/k8s/kube-controller-manager.pem \
--client-key=certs/k8s/kube-controller-manager-key.pem \
--embed-certs=true \
--kubeconfig=certs/k8s/controller-manager.conf

kubernetes/server/bin/kubectl config set-context default \
--cluster=kubernetes \
--user=system:kube-controller-manager \
--kubeconfig=certs/k8s/controller-manager.conf

kubernetes/server/bin/kubectl config use-context default \
--kubeconfig=certs/k8s/controller-manager.conf

# For scheduler
kubernetes/server/bin/kubectl config set-cluster kubernetes \
--certificate-authority=certs/k8s/k8s-ca.pem \
--embed-certs=true \
--server=${KUBE_APISERVER} \
--kubeconfig=certs/k8s/scheduler.conf

kubernetes/server/bin/kubectl config set-credentials system:kube-scheduler \
--client-certificate=certs/k8s/kube-scheduler.pem \
--client-key=certs/k8s/kube-scheduler-key.pem \
--embed-certs=true \
--kubeconfig=certs/k8s/scheduler.conf

kubernetes/server/bin/kubectl config set-context default \
--cluster=kubernetes \
--user=system:kube-scheduler \
--kubeconfig=certs/k8s/scheduler.conf

kubernetes/server/bin/kubectl config use-context default \
--kubeconfig=certs/k8s/scheduler.conf

#Check the output in `certs/k8s`
ls certs/k8s
# admin.conf               kube-admin-csr.json          kube-controller-manager-csr.json  kubelet-etcd-client-csr.json  kube-scheduler.csr
# bootstrap.conf           kube-admin-key.pem           kube-controller-manager-key.pem   kubelet-etcd-client-key.pem   kube-scheduler-csr.json
# controller-manager.conf  kube-admin.pem               kube-controller-manager.pem       kubelet-etcd-client.pem       kube-scheduler-key.pem
# k8s-ca.csr               kube-apiserver.csr           kube-front-proxy-client.csr       kubelet-master.conf           kube-scheduler.pem
# k8s-ca-csr.json          kube-apiserver-csr.json      kube-front-proxy-client-csr.json  kubelet-master.csr            sa.key
# k8s-ca-key.pem           kube-apiserver-key.pem       kube-front-proxy-client-key.pem   kubelet-master-csr.json       sa.pub
# k8s-ca.pem               kube-apiserver.pem           kube-front-proxy-client.pem       kubelet-master-key.pem        scheduler.conf
# kube-admin.csr           kube-controller-manager.csr  kubelet-etcd-client.csr           kubelet-master.pem            token.csv
```

- Distribute certificate and kubeconfig file to master and nodes

```bash

#For non-master node, without kubeconfig file,only copy bootstrap.conf
for node in ${K8S_NODES} ;do
  echo Copy node certificate file to $node
  ssh ${node} mkdir -p /etc/kubernetes/pki
  for files in \
    certs/etcd/etcd-ca.pem \
    certs/k8s/kubelet-etcd-client*.pem \
    certs/k8s/k8s-ca*.pem ;\
  do
    scp $files ${node}:/etc/kubernetes/pki
  done
  scp certs/k8s/bootstrap.conf ${node}:/etc/kubernetes
done

#For master node,which has --kubeconfig file copied
echo Copy node certificate file to ${KUBE_MASTER_SERV_01}[master]
for files in \
  certs/k8s/*.pem \
  certs/k8s/sa.* \
  certs/etcd/etcd-ca.pem ; \
do
  scp $files ${KUBE_MASTER_SERV_01}:/etc/kubernetes/pki
done
scp certs/k8s/*.conf ${KUBE_MASTER_SERV_01}:/etc/kubernetes
scp certs/k8s/token.csv ${KUBE_MASTER_SERV_01}:/etc/kubernetes

```

## 10. Setup `apiserver` `controller-manager` `scheduler` static pods yaml configs

```bash
cd
mkdir kubernetes/manifests -p
```

- For `apiserver` pod

```bash
cat > kubernetes/manifests/kube-apiserver.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ""
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  hostNetwork: true
  containers :
  - name: kube-apiserver
    image: ${PRIVATE_REGISTRY}/kube-apiserver:${K8S_BASE_VER}
    command:
      - kube-apiserver
      - --v=0
      - --logtostderr=true
      - --allow-privileged=true
      - --bind-address=0.0.0.0
      - --secure-port=${KUBE_APISERVER_PORT}
      - --insecure-port=0
      - --advertise-address=${KUBE_APISERVER_IP_01}
      - --service-cluster-ip-range=${KUBE_CLUSTER_SERVICE_CIDR}
      - --etcd-servers=https://${ETCD1_IP}:${ETCD_CLIENT_PORT},https://${ETCD2_IP}:${ETCD_CLIENT_PORT},https://${ETCD3_IP}:${ETCD_CLIENT_PORT}
      - --etcd-cafile=/etc/kubernetes/pki/etcd-ca.pem
      - --etcd-certfile=/etc/kubernetes/pki/kubelet-etcd-client.pem
      - --etcd-keyfile=/etc/kubernetes/pki/kubelet-etcd-client-key.pem
      - --client-ca-file=/etc/kubernetes/pki/k8s-ca.pem
      - --tls-cert-file=/etc/kubernetes/pki/kube-apiserver.pem
      - --tls-private-key-file=/etc/kubernetes/pki/kube-apiserver-key.pem
      - --kubelet-client-certificate=/etc/kubernetes/pki/kube-apiserver.pem
      - --kubelet-client-key=/etc/kubernetes/pki/kube-apiserver-key.pem
      - --service-account-key-file=/etc/kubernetes/pki/sa.pub
      - --token-auth-file=/etc/kubernetes/token.csv
      - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
      - --admission-control=Initializers,NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota
      - --authorization-mode=Node,RBAC
      - --enable-bootstrap-token-auth=true
      - --requestheader-client-ca-file=/etc/kubernetes/pki/k8s-ca.pem
      - --proxy-client-cert-file=/etc/kubernetes/pki/kube-front-proxy-client.pem
      - --proxy-client-key-file=/etc/kubernetes/pki/kube-front-proxy-client-key.pem
      #- --requestheader-allowed-names=aggregator
      - --requestheader-group-headers=X-Remote-Group
      - --requestheader-extra-headers-prefix=X-Remote-Extra-
      - --requestheader-username-headers=X-Remote-User
      - --event-ttl=1h
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: ${KUBE_APISERVER_PORT}
        scheme: HTTPS
      initialDelaySeconds: 15
      timeoutSeconds: 15
    resources:
      requests:
        cpu: 250m
    volumeMounts:
    - mountPath: /var/log/kubernetes
      name: k8s-audit-log
    - mountPath: /etc/kubernetes/pki
      name: k8s-certs
      readOnly: true
    - mountPath: /etc/kubernetes/token.csv
      name: token-csv
      readOnly: true
  volumes:
  - hostPath:
      path: /var/log/kubernetes
      type: DirectoryOrCreate
    name: k8s-audit-log
  - hostPath:
      path: /etc/kubernetes/pki
      type: DirectoryOrCreate
    name: k8s-certs
  - hostPath:
      path: /etc/kubernetes/token.csv
      type: FileOrCreate
    name: token-csv
EOF
```
- For `controller manager`

```bash
cat > kubernetes/manifests/kube-controller-manager.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ""
  labels:
    component: kube-controller-manager
    tier: control-plane
  name: kube-controller-manager
  namespace: kube-system
spec:
  hostNetwork: true
  containers:
  - name: kube-controller-manager
    image: ${PRIVATE_REGISTRY}/kube-controller-manager:${K8S_BASE_VER}
    command:
      - kube-controller-manager
      - --v=0
      - --logtostderr=true
      - --address=127.0.0.1
      - --root-ca-file=/etc/kubernetes/pki/k8s-ca.pem
      - --cluster-signing-cert-file=/etc/kubernetes/pki/k8s-ca.pem
      - --cluster-signing-key-file=/etc/kubernetes/pki/k8s-ca-key.pem
      - --service-account-private-key-file=/etc/kubernetes/pki/sa.key
      - --kubeconfig=/etc/kubernetes/controller-manager.conf
      - --leader-elect=true
      - --use-service-account-credentials=true
      - --node-monitor-grace-period=40s
      - --node-monitor-period=5s
      - --pod-eviction-timeout=2m0s
      - --controllers=*,bootstrapsigner,tokencleaner
      - --allocate-node-cidrs=true
      - --cluster-cidr=${KUBE_CLUSTER_POD_CIDR}
      - --node-cidr-mask-size=24
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10252
        scheme: HTTP
      initialDelaySeconds: 15
      timeoutSeconds: 15
    resources:
      requests:
        cpu: 200m
    volumeMounts:
    - mountPath: /etc/kubernetes/pki
      name: k8s-certs
      readOnly: true
    - mountPath: /etc/ssl/certs
      name: ca-certs
      readOnly: true
    - mountPath: /etc/kubernetes/controller-manager.conf
      name: kubeconfig
      readOnly: true
    - mountPath: /usr/libexec/kubernetes/kubelet-plugins/volume/exec
      name: flexvolume-dir
  volumes:
  - hostPath:
      path: /etc/kubernetes/pki
      type: DirectoryOrCreate
    name: k8s-certs
  - hostPath:
      path: /etc/ssl/certs
      type: DirectoryOrCreate
    name: ca-certs
  - hostPath:
      path: /etc/kubernetes/controller-manager.conf
      type: FileOrCreate
    name: kubeconfig
  - hostPath:
      path: /usr/libexec/kubernetes/kubelet-plugins/volume/exec
      type: DirectoryOrCreate
    name: flexvolume-dir
EOF
```

- For `scheduler`

```bash
cat > kubernetes/manifests/kube-scheduler.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ""
  labels:
    component: kube-scheduler
    tier: control-plane
  name: kube-scheduler
  namespace: kube-system
spec:
  hostNetwork: true
  containers:
  - name: kube-scheduler
    image: ${PRIVATE_REGISTRY}/kube-scheduler:${K8S_BASE_VER}
    command:
      - kube-scheduler
      - --v=0
      - --logtostderr=true
      - --address=127.0.0.1
      - --leader-elect=true
      - --kubeconfig=/etc/kubernetes/scheduler.conf
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10251
        scheme: HTTP
      initialDelaySeconds: 15
      timeoutSeconds: 15
    resources:
      requests:
        cpu: 100m
    volumeMounts:
    - mountPath: /etc/kubernetes/pki
      name: k8s-certs
      readOnly: true
    - mountPath: /etc/kubernetes/scheduler.conf
      name: kubeconfig
      readOnly: true
  volumes:
  - hostPath:
      path: /etc/kubernetes/pki
      type: DirectoryOrCreate
    name: k8s-certs
  - hostPath:
      path: /etc/kubernetes/scheduler.conf
      type: FileOrCreate
    name: kubeconfig
EOF
```
- Distribute to master node

```bash
ssh ${KUBE_MASTER_SERV_01} mkdir -p /etc/kubernetes/manifests
scp kubernetes/manifests/*.yaml ${KUBE_MASTER_SERV_01}:/etc/kubernetes/manifests
```

## 11. Setup and boot the `kubelet` & **cluster** with systemd service

```bash
cd
mkdir kubernetes/systemd/ -p

cat > kubernetes/systemd/kubelet.service << EOF
[Unit]
Description=kubelet: The Kubernetes Node Agent
Documentation=http://kubernetes.io/docs/

[Service]
ExecStart=/usr/local/bin/kubelet
Restart=on-failure
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

cat > kubernetes/systemd/10-kubelet.conf << EOF
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--address=0.0.0.0 --port=10250 --kubeconfig=/etc/kubernetes/kubelet-master.conf --bootstrap-kubeconfig=/etc/kubernetes/bootstrap.conf"
Environment="KUBE_LOGTOSTDERR=--logtostderr=true --v=0"
Environment="KUBELET_SYSTEM_PODS_ARGS=--pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true --anonymous-auth=false"
Environment="KUBELET_POD_CONTAINER=--pod-infra-container-image=${PRIVATE_REGISTRY}/${K8S_PAUSE_IMG_NAME}:${K8S_PAUSE_VER}"
Environment="KUBELET_NETWORK_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"
Environment="KUBELET_DNS_ARGS=--cluster-dns=${CLUSTER_DNS_IP} --cluster-domain=cluster.local"
Environment="KUBELET_AUTHZ_ARGS=--authorization-mode=Webhook --client-ca-file=/etc/kubernetes/pki/k8s-ca.pem"
Environment="KUBELET_CADVISOR_ARGS=--cadvisor-port=0"
Environment="KUBELET_CERTIFICATE_ARGS=--rotate-certificates=true --cert-dir=/var/lib/kubelet/pki"
Environment="KUBELET_EXTRA_ARGS=--fail-swap-on=false --serialize-image-pulls=false"
Environment="KUBE_NODE_LABEL=--node-labels=node-role.kubernetes.io/master=true"
#kubelet cgroup-driver MUST be consistence with docker daemon
Environment="KUBELET_CGROUP=--cgroup-driver=systemd"
ExecStart=
ExecStart=/usr/local/bin/kubelet \$KUBELET_KUBECONFIG_ARGS \$KUBE_LOGTOSTDERR \$KUBELET_POD_CONTAINER \$KUBELET_SYSTEM_PODS_ARGS \$KUBELET_NETWORK_ARGS \$KUBELET_DNS_ARGS \$KUBELET_AUTHZ_ARGS \$KUBELET_EXTRA_ARGS \$KUBE_NODE_LABEL \$KUBELET_CGROUP
EOF

for node in ${K8S_NODES} ;do
  echo Prepare for $node
  ssh ${node} mkdir -p /etc/systemd/system/kubelet.service.d
  scp kubernetes/systemd/10-kubelet.conf ${node}:/etc/systemd/system/kubelet.service.d/
  scp kubernetes/systemd/kubelet.service ${node}:/lib/systemd/system/kubelet.service
  ssh ${node} "systemctl stop kubelet && systemctl disable kubelet"
  ssh ${node} "systemctl enable kubelet && systemctl start kubelet"
done
```

## 12. Setup admin cli environment and check cluster status
- Setup the admin cli environment with `amdin.con` file

```bash
ssh ${KUBE_MASTER_SERV_01} mkdir ~/.kube
ssh ${KUBE_MASTER_SERV_01} cp /etc/kubernetes/admin.conf ~/.kube/config
ssh ${KUBE_MASTER_SERV_01} kubectl get cs
ssh ${KUBE_MASTER_SERV_01} kubectl get node
ssh ${KUBE_MASTER_SERV_01} kubectl -n kube-system get po
```
- The run out result should be like this:

```bash
ssh ${KUBE_MASTER_SERV_01} kubectl get cs
# NAME                 STATUS    MESSAGE              ERROR
# controller-manager   Healthy   ok
# scheduler            Healthy   ok
# etcd-2               Healthy   {"health": "true"}
# etcd-0               Healthy   {"health": "true"}
# etcd-1               Healthy   {"health": "true"}
ssh ${KUBE_MASTER_SERV_01} kubectl get node
# NAME      STATUS     ROLES     AGE       VERSION
# jin       NotReady   master    1m        v1.9.0
ssh ${KUBE_MASTER_SERV_01} kubectl -n kube-system get po
# NAME                          READY     STATUS    RESTARTS   AGE
# kube-apiserver-jin            1/1       Running   0          40s
# kube-controller-manager-jin   1/1       Running   0          33s
# kube-scheduler-jin            1/1       Running   0          38s
```

## 13. Authorize other K8s nodes joining the cluster
- Add TLS Bootstrapping ClusterRoleBinding to cluster

```bash
ssh ${KUBE_MASTER_SERV_01} kubectl create \
  clusterrolebinding kubelet-bootstrap \
  --clusterrole=system:node-bootstrapper \
  --user=kubelet-bootstrap
```

- Check pending node certificate CSR requests

```bash
ssh ${KUBE_MASTER_SERV_01} kubectl get csr
# NAME                                                   AGE       REQUESTOR           CONDITION
# node-csr-gO_HGaj_cJdNzjx3AxjmMtKBNfiBRUdIfBn0HdDyWtM   8s        kubelet-bootstrap   Pending
# node-csr-nrYBLZlNzplcs5pW81BvKTzzenqzXCSQ1IddYFs0BLY   8s        kubelet-bootstrap   Pending
# node-csr-t6TpFDn-uaou9RWRBs0cAXd9V2RIDgretORPWkM0--E   7s        kubelet-bootstrap   Pending
```

- Authorize node CSR requests
```bash
ssh ${KUBE_MASTER_SERV_01} "kubectl get csr | awk '/Pending/ {print \$1}' | xargs kubectl certificate approve"
# certificatesigningrequest "node-csr-gO_HGaj_cJdNzjx3AxjmMtKBNfiBRUdIfBn0HdDyWtM" approved
# certificatesigningrequest "node-csr-nrYBLZlNzplcs5pW81BvKTzzenqzXCSQ1IddYFs0BLY" approved
# certificatesigningrequest "node-csr-t6TpFDn-uaou9RWRBs0cAXd9V2RIDgretORPWkM0--E" approved

ssh ${KUBE_MASTER_SERV_01} kubectl get nodes
# NAME      STATUS     ROLES     AGE       VERSION
# huo       NotReady   master    34s       v1.9.0
# jin       NotReady   master    3m        v1.9.0
# mu        NotReady   master    33s       v1.9.0
# shui      NotReady   master    34s       v1.9.0
```

## 14. Setup `kube-proxy` addon
Which shall be deployed ahead of `Calico` ,otherwise the `Calico` components would stuck in creating container status.

```bash
cd
mkdir -p kubernetes/addons

#Generate kube proxy certificate
cat <<EOF > certs/k8s/kube-proxy-csr.json
{
  "CN":"system:kube-proxy",
  "key":{
    "algo":"rsa",
    "size":2048
  },
  "names":[
    {
      "O":"system:kube-proxy"
    }
  ]
}
EOF

cfssl gencert -ca=certs/k8s/k8s-ca.pem \
  -ca-key=certs/k8s/k8s-ca-key.pem \
  -config=certs/ca-config.json \
  -profile=default \
  certs/k8s/kube-proxy-csr.json | cfssljson -bare certs/k8s/kube-proxy

#Generate kubeconfig for proxy
kubernetes/server/bin/kubectl config set-cluster kubernetes \
--certificate-authority=certs/k8s/k8s-ca.pem \
--embed-certs=true \
--server=${KUBE_APISERVER} \
--kubeconfig=certs/k8s/kube-proxy.conf

kubernetes/server/bin/kubectl config set-credentials system:kube-proxy \
--client-certificate=certs/k8s/kube-scheduler.pem \
--client-key=certs/k8s/kube-scheduler-key.pem \
--embed-certs=true \
--kubeconfig=certs/k8s/kube-proxy.conf

kubernetes/server/bin/kubectl config set-context default \
--cluster=kubernetes \
--user=system:kube-proxy \
--kubeconfig=certs/k8s/kube-proxy.conf

kubernetes/server/bin/kubectl config use-context default \
--kubeconfig=certs/k8s/kube-proxy.conf

#Generate kube-proxy yaml
cat > kubernetes/addons/kube-proxy.yml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-proxy
  labels:
    k8s-app: kube-proxy
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
  namespace: kube-system
---
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: kube-proxy
  labels:
    k8s-app: kube-proxy
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: kube-proxy
  templateGeneration: 1
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        k8s-app: kube-proxy
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      serviceAccountName: kube-proxy
      hostNetwork: true
      containers:
      - name: kube-proxy
        image: ${PRIVATE_REGISTRY}/${K8S_PROXY_IMG_NAME}:${K8S_PROXY_VER}
        command:
        - kube-proxy
        - --v=0
        - --logtostderr=true
        - --kubeconfig=/run/kube-proxy.conf
        - --cluster-cidr=${KUBE_CLUSTER_POD_CIDR}
        - --proxy-mode=iptables
        imagePullPolicy: IfNotPresent
        securityContext:
          privileged: true
        volumeMounts:
        - mountPath: /run/kube-proxy.conf
          name: kubeconfig
          readOnly: true
        - mountPath: /etc/kubernetes/pki
          name: k8s-certs
          readOnly: true
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      volumes:
      - hostPath:
          path: /etc/kubernetes/kube-proxy.conf
          type: FileOrCreate
        name: kubeconfig
      - hostPath:
          path: /etc/kubernetes/pki
          type: DirectoryOrCreate
        name: k8s-certs
EOF

#Distribute certs and config file to nodes
for node in ${K8S_NODES} ; do
  scp certs/k8s/kube-proxy*.pem ${node}:/etc/kubernetes/pki
  scp certs/k8s/kube-proxy.conf ${node}:/etc/kubernetes/
done
ssh ${KUBE_MASTER_SERV_01} "mkdir -p /etc/kubernetes/addons"
scp kubernetes/addons/kube-proxy.yml ${KUBE_MASTER_SERV_01}:/etc/kubernetes/addons

#Apply yaml to cluster
ssh ${KUBE_MASTER_SERV_01} kubectl apply -f /etc/kubernetes/addons/kube-proxy.yml

#Check status
ssh ${KUBE_MASTER_SERV_01} kubectl -n kube-system get po -l k8s-app=kube-proxy
# NAME               READY     STATUS    RESTARTS   AGE
# kube-proxy-drfbj   1/1       Running   0          45s
# kube-proxy-lx6sx   1/1       Running   0          45s
# kube-proxy-m44b8   1/1       Running   0          45s
# kube-proxy-nb2sw   1/1       Running   0          44s

```

## 15. `Calico` Standard Hosted Install
- Generate certificate for calico-etcd-client

```bash
cd
mkdir certs/calico -p
cat <<EOF > certs/calico/calico-etcd-client-csr.json
{
  "CN":"calico-nodes",
  "key":{
    "algo":"rsa",
    "size":2048
  }
}
EOF

cfssl gencert -ca=certs/etcd/etcd-ca.pem \
  -ca-key=certs/etcd/etcd-ca-key.pem \
  -config=certs/ca-config.json \
  -profile=default \
  certs/calico/calico-etcd-client-csr.json | cfssljson -bare certs/calico/calico-etcd-client

```
- Modify preset yaml pattern
Setup Calico network in IPIP `OFF` mode,the pattern is fetched from [Standard Hosted Install](https://docs.projectcalico.org/v2.6/getting-started/kubernetes/installation/hosted/hosted) - [calico.yaml](https://docs.projectcalico.org/v2.6/getting-started/kubernetes/installation/hosted/calico.yaml)

```bash
export ETCD_CA_BASE64=`cat ${HOME}/certs/etcd/etcd-ca.pem|base64 -w 0`
export ETCD_CERT_BASE64=`cat ${HOME}/certs/calico/calico-etcd-client.pem|base64 -w 0`
export ETCD_KEY_BASE64=`cat ${HOME}/certs/calico/calico-etcd-client-key.pem|base64 -w 0`
sed "s#http://127.0.0.1:2379#$ETCD_ENDPOINTS#" \
all-in-one/configs/calico-deploy.yaml.conf |\
sed 's/etcd_ca: ""   #/etcd_ca: /g' |\
sed 's/etcd_cert: "" #/etcd_cert: /g' |\
sed 's/etcd_key: ""  #/etcd_key: /g' |\
sed "s/  # etcd-key: null/  etcd-key: $ETCD_KEY_BASE64/" |\
sed "s/  # etcd-cert: null/  etcd-cert: $ETCD_CERT_BASE64/" |\
sed "s/  # etcd-ca: null/  etcd-ca: $ETCD_CA_BASE64/" |\
sed "s#quay.io/calico/${CALICO_NODE_IMAGE_NAME}:${CALICO_NODE_VER}#${PRIVATE_REGISTRY}/calico/${CALICO_NODE_IMAGE_NAME}:${CALICO_NODE_VER}#g" |\
sed "s#192.168.0.0/16#${KUBE_CLUSTER_POD_CIDR}#" |\
sed "s#quay.io/calico/${CALICO_CNI_IMAGE_NAME}:${CALICO_CNI_VER}#${PRIVATE_REGISTRY}/calico/${CALICO_CNI_IMAGE_NAME}:${CALICO_CNI_VER}#g" |\
sed "s#quay.io/calico/${CALICO_CONTROLLER_IMG_NAME}:${CALICO_CONTROLLER_VER}#${PRIVATE_REGISTRY}/calico/${CALICO_CONTROLLER_IMG_NAME}:${CALICO_CONTROLLER_VER}#g" |\
sed "/CALICO_IPV4POOL_IPIP/{n;d}" |\
sed '/CALICO_IPV4POOL_IPIP/a\              value: "off"' > \
all-in-one/configs/calico-deploy.yaml

#Distribute config file to master node
scp all-in-one/configs/calico-deploy.yaml ${KUBE_MASTER_SERV_01}:/etc/kubernetes/addons
scp all-in-one/configs/calico-rbac.yaml ${KUBE_MASTER_SERV_01}:/etc/kubernetes/addons
ssh ${KUBE_MASTER_SERV_01} kubectl apply -f /etc/kubernetes/addons/calico-rbac.yaml
ssh ${KUBE_MASTER_SERV_01} kubectl apply -f /etc/kubernetes/addons/calico-deploy.yaml

#Check the calico pod status
ssh ${KUBE_MASTER_SERV_01} kubectl -n kube-system get po
# NAME                                       READY     STATUS    RESTARTS   AGE
# calico-kube-controllers-54747bbfbf-rx48m   1/1       Running   0          2m
# calico-node-2h6r8                          2/2       Running   0          2m
# calico-node-gfwzf                          2/2       Running   0          2m
# calico-node-gvr7v                          2/2       Running   0          2m
# calico-node-lf4zp                          2/2       Running   0          2m
# kube-apiserver-jin                         1/1       Running   0          16m
# kube-controller-manager-jin                1/1       Running   0          16m
# kube-proxy-drfbj                           1/1       Running   0          12m
# kube-proxy-lx6sx                           1/1       Running   0          12m
# kube-proxy-m44b8                           1/1       Running   0          12m
# kube-proxy-nb2sw                           1/1       Running   0          12m
# kube-scheduler-jin                         1/1       Running   0          16m
```

## 16. Setup `kube-dns` addon

```bash
cat > kubernetes/addons/kube-dns.yml << EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-dns
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
  namespace: kube-system
---
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: ${CLUSTER_DNS_IP}
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: kube-dns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  strategy:
    rollingUpdate:
      maxSurge: 10%
      maxUnavailable: 0
  selector:
    matchLabels:
      k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      dnsPolicy: Default
      serviceAccountName: kube-dns
      tolerations:
      - key: "CriticalAddonsOnly"
        operator: "Exists"
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      volumes:
      - name: kube-dns-config
        configMap:
          name: kube-dns
          optional: true
      containers:
      - name: kubedns
        image: ${PRIVATE_REGISTRY}/${K8S_KUBE_DNS_IMG_NAME}:${K8S_DNS_VER}
        resources:
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        livenessProbe:
          httpGet:
            path: /healthcheck/kubedns
            port: 10054
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
        readinessProbe:
          httpGet:
            path: /readiness
            port: 8081
            scheme: HTTP
          initialDelaySeconds: 3
          timeoutSeconds: 5
        args:
        - "--domain=cluster.local"
        - --dns-port=10053
        - --v=2
        env:
        - name: PROMETHEUS_PORT
          value: "10055"
        ports:
        - containerPort: 10053
          name: dns-local
          protocol: UDP
        - containerPort: 10053
          name: dns-tcp-local
          protocol: TCP
        - containerPort: 10055
          name: metrics
          protocol: TCP
        volumeMounts:
        - name: kube-dns-config
          mountPath: /kube-dns-config
      - name: dnsmasq
        image: ${PRIVATE_REGISTRY}/${K8S_DNSMASQ_IMG_NAME}:${K8S_DNS_VER}
        livenessProbe:
          httpGet:
            path: /healthcheck/dnsmasq
            port: 10054
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
        args:
        - "-v=2"
        - "-logtostderr"
        - "-configDir=/etc/k8s/dns/dnsmasq-nanny"
        - "-restartDnsmasq=true"
        - "--"
        - "-k"
        - "--cache-size=1000"
        - "--log-facility=-"
        - "--server=/cluster.local/127.0.0.1#10053"
        - "--server=/in-addr.arpa/127.0.0.1#10053"
        - "--server=/ip6.arpa/127.0.0.1#10053"
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        resources:
          requests:
            cpu: 150m
            memory: 20Mi
        volumeMounts:
        - name: kube-dns-config
          mountPath: /etc/k8s/dns/dnsmasq-nanny
      - name: sidecar
        image: ${PRIVATE_REGISTRY}/${K8S_SIDECAR_IMG_NAME}:${K8S_DNS_VER}
        livenessProbe:
          httpGet:
            path: /metrics
            port: 10054
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
        args:
        - "--v=2"
        - "--logtostderr"
        - "--probe=kubedns,127.0.0.1:10053,kubernetes.default.svc.cluster.local,5,A"
        - "--probe=dnsmasq,127.0.0.1:53,kubernetes.default.svc.cluster.local,5,A"
        ports:
        - containerPort: 10054
          name: metrics
          protocol: TCP
        resources:
          requests:
            memory: 20Mi
            cpu: 10m
EOF

#Distribute certs and config file to nodes
ssh ${KUBE_MASTER_SERV_01} mkdir -p /etc/kubernetes/addons
scp kubernetes/addons/kube-dns.yml ${KUBE_MASTER_SERV_01}:/etc/kubernetes/addons

#Apply yaml to cluster
ssh ${KUBE_MASTER_SERV_01} kubectl apply -f /etc/kubernetes/addons/kube-dns.yml

#Check DNS status with cluster service IP
ssh ${KUBE_MASTER_SERV_01} kubectl -n kube-system get po -l k8s-app=kube-dns -o wide
# NAME                       READY     STATUS    RESTARTS   AGE       IP             NODE
# kube-dns-c84c85bb4-svv9v   3/3       Running   0          2m        172.17.95.65   mu
```

## 17. Check the whole cluster status
That's all for the cluster deployments, now you can check the overall status in the cluster

```bash
ssh ${KUBE_MASTER_SERV_01} kubectl -n kube-system get all
# NAME             DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
# ds/calico-node   4         4         4         4            4           <none>          5m
# ds/kube-proxy    4         4         4         4            4           <none>          16m
#
# NAME                              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
# deploy/calico-kube-controllers    1         1         1            1           5m
# deploy/calico-policy-controller   0         0         0            0           5m
# deploy/kube-dns                   1         1         1            1           1m
#
# NAME                                     DESIRED   CURRENT   READY     AGE
# rs/calico-kube-controllers-54747bbfbf    1         1         1         5m
# rs/calico-policy-controller-6d964cc58f   0         0         0         5m
# rs/kube-dns-c84c85bb4                    1         1         1         1m
#
# NAME             DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
# ds/calico-node   4         4         4         4            4           <none>          5m
# ds/kube-proxy    4         4         4         4            4           <none>          16m
#
# NAME                              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
# deploy/calico-kube-controllers    1         1         1            1           5m
# deploy/calico-policy-controller   0         0         0            0           5m
# deploy/kube-dns                   1         1         1            1           1m
#
# NAME                                     DESIRED   CURRENT   READY     AGE
# rs/calico-kube-controllers-54747bbfbf    1         1         1         5m
# rs/calico-policy-controller-6d964cc58f   0         0         0         5m
# rs/kube-dns-c84c85bb4                    1         1         1         1m
#
# NAME                                          READY     STATUS    RESTARTS   AGE
# po/calico-kube-controllers-54747bbfbf-rx48m   1/1       Running   0          5m
# po/calico-node-2h6r8                          2/2       Running   0          5m
# po/calico-node-gfwzf                          2/2       Running   0          5m
# po/calico-node-gvr7v                          2/2       Running   0          5m
# po/calico-node-lf4zp                          2/2       Running   0          5m
# po/kube-apiserver-jin                         1/1       Running   0          19m
# po/kube-controller-manager-jin                1/1       Running   0          19m
# po/kube-dns-c84c85bb4-svv9v                   3/3       Running   0          1m
# po/kube-proxy-drfbj                           1/1       Running   0          16m
# po/kube-proxy-lx6sx                           1/1       Running   0          16m
# po/kube-proxy-m44b8                           1/1       Running   0          16m
# po/kube-proxy-nb2sw                           1/1       Running   0          15m
# po/kube-scheduler-jin                         1/1       Running   0          19m
#
# NAME           TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)         AGE
# svc/kube-dns   ClusterIP   172.16.0.10   <none>        53/UDP,53/TCP   1m
```

### References
- [Standard Hosted Install](https://docs.projectcalico.org/v2.6/getting-started/kubernetes/installation/hosted/hosted)

- [Kubernetes 1.8.x 全手动安装教程](https://www.kubernetes.org.cn/3096.html)
