---
layout: post
title: "Deploy K8s + Calico network in air-gapped environment"
date: 2017-12-18 14:16:01 +0800
tags: ["Kubenetes","Calico"]
---

## Pre-defined environment vars
```bash
export TARGET_USER=highland
export TIANJIN_IP=192.168.122.155
export SHANGHAI_IP=192.168.122.80
export GUANGZHOU_IP=192.168.122.30
export XINING_IP=192.168.122.86
export NODES="shanghai guangzhou tianjin xining"
export ETCD_CLIENT_PORT=2379
export ETCD_PEER_PORT=2380
export ETCD1_IP=$SHANGHAI_IP
export ETCD2_IP=$GUANGZHOU_IP
export ETCD3_IP=$TIANJIN_IP
export ETCD_IPS="($ETCD1_IP $ETCD2_IP $ETCD3_IP)"
export ETCD_NODES="shanghai guangzhou tianjin"
export PRIVATE_REGISTRY_SERV=shanghai
export PRIVATE_REGISTRY_SERV_IP=$SHANGHAI_IP
export PRIVATE_REGISTRY=${PRIVATE_REGISTRY_SERV}:443
export KUBE_MASTER_SERV_01="tianjin"
export KUBE_MASTER_SERV_IP_01=$TIANJIN_IP
export KUBE_APISERVER_IP_01=$KUBE_MASTER_SERV_IP_01
export KUBE_APISERVER=https://$KUBE_APISERVER_IP_01:6443
export KUBE_CLUSTER_SERVICE_IP=172.16.0.1
export KUBE_CLUSTER_SERVICE_CIDR=172.16.0.1/16
export KUBE_CLUSTER_POD_CIDR=172.17.0.1/16
export CLUSTER_DNS=172.16.0.10
export K8S_VER=v1.9.0
export GCR_IMAGES="kube-aggregator kube-controller-manager kube-proxy kube-scheduler kube-apiserver"

#TODO Make preset envs through source script
```

## 1. Install Docker
Download docker package and dependencies
```bash
mkdir ${HOME}/rpms
yum update && yum install -y --downloadonly  --downloaddir=${HOME}/rpms docker
```
Install docker on target hosts
```bash
# Scp docker rpm packages to each nodes, passwdless environment must be set previously for root user
# Execute current command as root
cd
./preset_env.sh
cd ~/rpms
for node in ${NODES} ; do
  ssh $node mkdir ~/rpms -p && scp *.rpm ${node}:~/rpms
  ssh $node rpm -ivh ~/rpms/*.rpm
  ssh $node groupadd docker
  ssh $node usermod -aG docker $TARGET_USER
  ssh $node setenforce 0
  ssh $node "sed -i 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/selinux/config"
  ssh $node systemctl stop firewalld
  ssh $node systemctl disable firewalld
  ssh $node "sed -i '/^ExecReload/i\ExecStartPost=\/sbin\/iptables -I FORWARD -s 0.0.0.0\/0 -j ACCEPT' /lib/systemd/system/docker.service"
  ssh $node systemctl disable docker
  ssh $node systemctl enable docker
  ssh $node systemctl stop docker
  ssh $node systemctl start docker
done
```
## 2. Setup private registry service
On Internet and local network both available host
```bash
cd
./preset_env.sh
docker pull registry && docker save docker.io/registry:latest -o docker-io.registry.tar
export CFSSL_URL="https://pkg.cfssl.org/R1.2"
curl -L "${CFSSL_URL}/cfssl_linux-amd64" -o /usr/local/bin/cfssl
curl -L "${CFSSL_URL}/cfssljson_linux-amd64" -o /usr/local/bin/cfssljson
chmod +x /usr/local/bin/cfssl*

scp docker-io.registry.tar ${PRIVATE_REGISTRY_SERV}:~/
ssh ${PRIVATE_REGISTRY_SERV} docker load -i ~/docker-io.registry.tar
ssh ${PRIVATE_REGISTRY_SERV} mkdir /etc/docker/ssl -p

mkdir certs/docker -p
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

#Make private registry trusted in the nodes
for node in ${NODES} ; do
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

## 3. Setup etcd daemon

- prepare certificates
```bash
cd
./preset_env.sh
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
  ssh ${node} mkdir -p /etc/etcd/ssl
  scp certs/etcd/* ${node}:/etc/etcd/ssl
done
```

- prepare binary and configs
```bash
cd
./preset_env.sh
export ETCD_URL="https://github.com/coreos/etcd/releases/download"
curl -L "${ETCD_URL}/v3.2.12/etcd-v3.2.12-linux-amd64.tar.gz" | tar -zx

let node_idx=0
for node in ${ETCD_NODES} ;do
  echo process node ${node}

  ssh ${node} systemctl stop etcd.service
  ssh ${node} systemctl disable etcd.service

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
ETCD_ADVERTISE_CLIENT_URLS=https://${ETCD_IPS[${node_idx}]}:${ETCD_CLIENT_PORT}
ETCD_INITIAL_ADVERTISE_PEER_URLS=https://${ETCD_IPS[${node_idx}]}:${ETCD_PEER_PORT}
ETCD_INITIAL_CLUSTER=shanghai=https://192.168.122.80:${ETCD_PEER_PORT},guangzhou=https://192.168.122.30:${ETCD_PEER_PORT},tianjin=https://192.168.122.155:${ETCD_PEER_PORT}
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

  ssh ${node} ETCDCTL_API=3 etcdctl --cacert=/etc/etcd/ssl/etcd-ca.pem --cert=/etc/etcd/ssl/etcd.pem --key=/etc/etcd/ssl/etcd-key.pem  --endpoints="https://${ETCD_IPS[${node_idx}]}:${ETCD_CLIENT_PORT}" endpoint health

  ssh ${node} ETCDCTL_API=3 etcdctl --cacert=/etc/etcd/ssl/etcd-ca.pem --cert=/etc/etcd/ssl/etcd.pem --key=/etc/etcd/ssl/etcd-key.pem  --endpoints="https://${ETCD_IPS[${node_idx}]}:${ETCD_CLIENT_PORT}" member list

  let node_idx=${node_idx}+1

done
```

## 4. Prepare K8s binaries and images
```bash
cd
./preset_env.sh
curl -L https://dl.k8s.io/v1.9.0/kubernetes-client-linux-amd64.tar.gz | tar -zx
curl -L https://dl.k8s.io/v1.9.0/kubernetes-server-linux-amd64.tar.gz | tar -zx
docker pull lhcalibur/pause-amd64:3.0
docker save docker.io/lhcalibur/pause-amd64:3.0 -o pause_amd64_3_0.tar
for node in ${NODES} ; do
  echo Process node ${node}.....
  scp kubernetes/server/bin/kubectl ${node}:/usr/local/bin
  scp kubernetes/server/bin/kubelet ${node}:/usr/local/bin
  ssh ${node} chmod +x /usr/local/bin/kube*
done

export gcr=gcr.io/google_containers

ssh ${PRIVATE_REGISTRY_SERV} mkdir ~/images
scp kubernetes/server/bin/*.tar ${PRIVATE_REGISTRY_SERV}:~/images
scp pause_amd64_3_0.tar ${PRIVATE_REGISTRY_SERV}:~/images

ssh ${PRIVATE_REGISTRY_SERV} docker load -i images/pause_amd64_3_0.tar  
ssh ${PRIVATE_REGISTRY_SERV} docker tag docker.io/lhcalibur/pause-amd64:3.0 ${PRIVATE_REGISTRY}/pause-amd64:3.0
ssh ${PRIVATE_REGISTRY_SERV} docker push  ${PRIVATE_REGISTRY}/pause-amd64:3.0
ssh ${PRIVATE_REGISTRY_SERV} docker rmi docker.io/lhcalibur/pause-amd64:3.0
ssh ${PRIVATE_REGISTRY_SERV} docker rmi  ${PRIVATE_REGISTRY}/pause-amd64:3.0

for img in ${GCR_IMAGES} ; do
  ssh ${PRIVATE_REGISTRY_SERV} docker load -i ~/images/${img}.tar
  ssh ${PRIVATE_REGISTRY_SERV} docker tag ${gcr}/${img}:${K8S_VER}  ${PRIVATE_REGISTRY}/${img}:${K8S_VER}
  ssh ${PRIVATE_REGISTRY_SERV} docker push  ${PRIVATE_REGISTRY}/${img}:${K8S_VER}
  ssh ${PRIVATE_REGISTRY_SERV} docker rmi ${gcr}/${img}:${K8S_VER}
  ssh ${PRIVATE_REGISTRY_SERV} docker rmi  ${PRIVATE_REGISTRY}/${img}:${K8S_VER}
done

ssh ${PRIVATE_REGISTRY_SERV} rm -rf images
```

## 5. Prepare certificate for K8s
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
      "C":"cn",
      "ST":"DreamState",
      "L":"DreamCity",
      "O":"kube-apiserver",
      "OU":"DreamTeam"
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
  },
  "names":[
    {
      "C":"cn",
      "ST":"DreamState",
      "L":"DreamCity",
      "O":"front-proxy-client",
      "OU":"DreamTeam"
    }
  ]
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
      "C":"cn",
      "ST":"DreamState",
      "L":"DreamCity",
      "O":"system:kube-controller-manager",
      "OU":"DreamTeam"
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
      "C":"cn",
      "ST":"DreamState",
      "L":"DreamCity",
      "O":"system:kube-scheduler",
      "OU":"DreamTeam"
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
      "C":"cn",
      "ST":"DreamState",
      "L":"DreamCity",
      "O":"system:nodes",
      "OU":"DreamTeam"
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

#For bootstrap token
export BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')
cat <<EOF > certs/k8s/token.csv
${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF

```

## 6. Prepare kubeconfig file for K8s
```bash
cd
# For bootstrap
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
```
- Check the output in `certs/k8s`
```bash
ls certs/k8s

admin.conf               kube-admin-csr.json          kube-controller-manager-csr.json  kubelet-master.csr       sa.key
bootstrap.conf           kube-admin-key.pem           kube-controller-manager-key.pem   kubelet-master-csr.json  sa.pub
controller-manager.conf  kube-admin.pem               kube-controller-manager.pem       kubelet-master-key.pem   scheduler.conf
k8s-ca.csr               kube-apiserver.csr           kube-front-proxy-client.csr       kubelet-master.pem       token.csv
k8s-ca-csr.json          kube-apiserver-csr.json      kube-front-proxy-client-csr.json  kube-scheduler.csr
k8s-ca-key.pem           kube-apiserver-key.pem       kube-front-proxy-client-key.pem   kube-scheduler-csr.json
k8s-ca.pem               kube-apiserver.pem           kube-front-proxy-client.pem       kube-scheduler-key.pem
kube-admin.csr           kube-controller-manager.csr  kubelet-master.conf               kube-scheduler.pem
```

- Distribute certificate and kubeconfig file to master and nodes
```bash
#For master node
ssh ${KUBE_MASTER_SERV_01} mkdir -p /etc/kubernetes/pki
scp certs/k8s/*.pem ${KUBE_MASTER_SERV_01}:/etc/kubernetes/pki
scp certs/k8s/sa.* ${KUBE_MASTER_SERV_01}:/etc/kubernetes/pki
scp certs/k8s/*.conf ${KUBE_MASTER_SERV_01}:/etc/kubernetes
scp certs/k8s/token.csv ${KUBE_MASTER_SERV_01}:/etc/kubernetes

for node in ${NODES} ;do
  ssh ${node} mkdir -p /etc/kubernets/pki
  #TODO copy necessary files to non-master nodes
done
```

## 7. Prepare kubelet runtime environment
```bash
for node in ${NODES} ;do
  echo Disable swap file on ${node}
  ssh ${node} swapoff -a
  ssh ${node} cp /etc/fstab /etc/fstab.with_swap
  echo orginal /etc/fstab file has been backup to etc/fstab.with_swap
  ssh ${node} sed -i '/swap/d' /etc/fstab
  #TODO copy necessary files to non-master nodes
done
```

## 8. Setup kubelet daemon
```bash
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
Environment="KUBELET_KUBECONFIG_ARGS=--address=0.0.0.0 --port=10250 --kubeconfig=/etc/kubernetes/kubelet-master.conf"
Environment="KUBE_LOGTOSTDERR=--logtostderr=true --v=0"
Environment="KUBELET_SYSTEM_PODS_ARGS=--pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true --anonymous-auth=false"
Environment="KUBELET_POD_CONTAINER=--pod-infra-container-image=${PRIVATE_REGISTRY}/pause-amd64:3.0"
Environment="KUBELET_NETWORK_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"
Environment="KUBELET_DNS_ARGS=--cluster-dns=${CLUSTER_DNS} --cluster-domain=cluster.local"
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

#ssh ${KUBE_MASTER_SERV_01} mkdir -p /var/log/kubernetes
ssh ${KUBE_MASTER_SERV_01} mkdir -p /etc/systemd/system/kubelet.service.d
scp kubernetes/systemd/10-kubelet.conf ${KUBE_MASTER_SERV_01}:/etc/systemd/system/kubelet.service.d/
scp kubernetes/systemd/kubelet.service ${KUBE_MASTER_SERV_01}:/lib/systemd/system/kubelet.service
ssh ${KUBE_MASTER_SERV_01} systemctl stop kubelet
ssh ${KUBE_MASTER_SERV_01} systemctl disable kubelet
ssh ${KUBE_MASTER_SERV_01} systemctl enable kubelet
ssh ${KUBE_MASTER_SERV_01} systemctl start kubelet
```

## 9. Setup apiserver\manager\scheduler static pods
```bash

mkdir kubernetes/manifests -p
#For apiserver pod
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
    image: ${PRIVATE_REGISTRY}/kube-apiserver:${K8S_VER}
    command:
      - kube-apiserver
      - --v=0
      - --logtostderr=true
      - --allow-privileged=true
      - --bind-address=0.0.0.0
      - --secure-port=6443
      - --insecure-port=0
      - --advertise-address=${KUBE_APISERVER_IP_01}
      - --service-cluster-ip-range=${KUBE_CLUSTER_SERVICE_CIDR}
      - --etcd-servers=https://${ETCD1_IP}:${ETCD_CLIENT_PORT}
      - --etcd-cafile=/etc/etcd/ssl/etcd-ca.pem
      - --etcd-certfile=/etc/etcd/ssl/etcd.pem
      - --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem
      - --client-ca-file=/etc/kubernetes/pki/k8s-ca.pem
      - --tls-cert-file=/etc/kubernetes/pki/kube-apiserver.pem
      - --tls-private-key-file=/etc/kubernetes/pki/kube-apiserver-key.pem
      - --kubelet-client-certificate=/etc/kubernetes/pki/kube-apiserver.pem
      - --kubelet-client-key=/etc/kubernetes/pki/kube-apiserver-key.pem
      - --service-account-key-file=/etc/kubernetes/pki/sa.pub
      - --token-auth-file=/etc/kubernetes/token.csv
      - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
      - --admission-control=Initializers,NamespaceLifecycle,LimitRanger,ServiceAccount,PersistentVolumeLabel,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota
      - --authorization-mode=Node,RBAC
      - --enable-bootstrap-token-auth=true
      - --requestheader-client-ca-file=/etc/kubernetes/pki/k8s-ca.pem
      - --proxy-client-cert-file=/etc/kubernetes/pki/kube-front-proxy-client.pem
      - --proxy-client-key-file=/etc/kubernetes/pki/kube-front-proxy-client-key.pem
      - --requestheader-allowed-names=aggregator
      - --requestheader-group-headers=X-Remote-Group
      - --requestheader-extra-headers-prefix=X-Remote-Extra-
      - --requestheader-username-headers=X-Remote-User
      - --event-ttl=1h
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 6443
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
    - mountPath: /etc/etcd/ssl
      name: etcd-ca-certs
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
  - hostPath:
      path: /etc/etcd/ssl
      type: DirectoryOrCreate
    name: etcd-ca-certs
EOF

#For controller manager
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
    image: ${PRIVATE_REGISTRY}/kube-controller-manager:${K8S_VER}
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

#For scheduler
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
    image: ${PRIVATE_REGISTRY}/kube-scheduler:${K8S_VER}
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

ssh ${KUBE_MASTER_SERV_01} mkdir -p /etc/kubernetes/manifests
scp kubernetes/manifests/*.yaml ${KUBE_MASTER_SERV_01}:/etc/kubernetes/manifests

```

## 10. Setup admin environment and check cluster status
```bash
mkdir ~/.kube
cp /etc/kubernetes/admin.conf ~/.kube/config
kubectl get cs
kubectl get node
kubectl -n kube-system get po
```
- The run out result should be like this:
```bash
[root@tianjin ~]# kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health": "true"}

[root@tianjin ~]# kubectl get node
NAME      STATUS     ROLES     AGE       VERSION
tianjin   NotReady   master    3m        v1.9.0

[root@tianjin ~]# kubectl -n kube-system get po
NAME                              READY     STATUS    RESTARTS   AGE
kube-apiserver-tianjin            1/1       Running   0          2m
kube-controller-manager-tianjin   1/1       Running   0          3m
kube-scheduler-tianjin            1/1       Running   0          2m
```
