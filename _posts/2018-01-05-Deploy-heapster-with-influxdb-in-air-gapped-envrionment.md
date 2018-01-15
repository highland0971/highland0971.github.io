---
layout: post
title: "Deploy heapster with influxdb and grafana in air-gapped environment"
date: 2018-01-05 16:48:01 +0800
tags: ["Kubenetes","Dashboard"]
---

This article describes deployment of heapster with influxdb and grafana in an air-gapped environments, based on the [pre-installed k8s cluster](https://highland0971.github.io/2017/12/29/Deploy-K8s-Calico-in-air-gapped-environment.html)

## 1. Pre-defined environment vars
We use the pre-defined vars according to article [Deploy K8s + Calico Network in air-gapped environment within 20 minutes](https://highland0971.github.io/2017/12/29/Deploy-K8s-Calico-in-air-gapped-environment.html), and add extra vars for dashboard.

```bash

#-----From [Deploy K8s + Calico Network in air-gapped environment within 20 minutes] ------
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
export REMOTE_REGISTRY=highland0971
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

#----- For dashboard ------
export K8S_DASHBOARD_IMG_NAME=kubernetes-dashboard-amd64
export K8S_DASHBOARD_VER=v1.8.1

#----- For heapster ------
export K8S_HEAPSTER_IMG_NAME=heapster-amd64
export K8S_HEAPSTER_VER=v1.5.0

#----- For heapster-influxdb ------
export K8S_HEAPSTER_INFLUXDB_IMG_NAME=heapster-influxdb-amd64
export K8S_HEAPSTER_INFLUXDB_VER=v1.3.3

#----- For heapster-grafana ------
export K8S_HEAPSTER_GRAFANA_IMG_NAME=heapster-grafana-amd64
export K8S_HEAPSTER_GRAFANA_VER=v4.4.3
```

## 1. Prepare required images and yaml config
- Download kubernetes-dashboard container image and push to private registry
We omitted the docker save & load procedure from Internet available host to air-gapped cluster `INSTALLATION_BASE`, you can refer artical revelent section in [pre-installed k8s cluster](https://highland0971.github.io/2017/12/29/Deploy-K8s-Calico-in-air-gapped-environment.html).

```bash
#For kubernetes dashboard image
#On Internet available host
mkdir -p all-in-one/images

for item in ${K8S_HEAPSTER_IMG_NAME}:${K8S_HEAPSTER_VER} ${K8S_HEAPSTER_INFLUXDB_IMG_NAME}:${K8S_HEAPSTER_INFLUXDB_VER} ${K8S_HEAPSTER_GRAFANA_IMG_NAME}:${K8S_HEAPSTER_GRAFANA_VER} ; do
  docker pull ${REMOTE_REGISTRY}/${item} && \
  docker tag ${REMOTE_REGISTRY}/${item} ${PRIVATE_REGISTRY}/${item} && \
  docker save ${PRIVATE_REGISTRY}/${item} -o all-in-one/images/${item}.tar
done

#On ${INSTALLATION_BASE}
ssh ${PRIVATE_REGISTRY_SERV} mkdir ~/images
for item in ${K8S_HEAPSTER_IMG_NAME}:${K8S_HEAPSTER_VER} ${K8S_HEAPSTER_INFLUXDB_IMG_NAME}:${K8S_HEAPSTER_INFLUXDB_VER} ${K8S_HEAPSTER_GRAFANA_IMG_NAME}:${K8S_HEAPSTER_GRAFANA_VER} ; do
  scp all-in-one/images/${item}.tar ${PRIVATE_REGISTRY_SERV}:~/images
  ssh ${PRIVATE_REGISTRY_SERV} docker load -i ~/images/${item}.tar
  ssh ${PRIVATE_REGISTRY_SERV} docker push ${PRIVATE_REGISTRY}/${item}
  ssh ${PRIVATE_REGISTRY_SERV} docker rmi ${PRIVATE_REGISTRY}/${item}
done
```

- Curl yaml from kubernetes-dashboard offical webset and modify according to your private registry location

```bash
curl -L "https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/heapster.yaml" -o all-in-one/configs/heapster.yaml.conf
curl -L "https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/grafana.yaml" -o all-in-one/configs/grafana.yaml.conf
curl -L "https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/influxdb.yaml" -o all-in-one/configs/influxdb.yaml.conf
curl -L "https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/rbac/heapster-rbac.yaml" -o all-in-one/configs/heapster-rbac.yaml

sed "s/k8s.gcr.io/${PRIVATE_REGISTRY}/g; s/v1.4.2/${K8S_HEAPSTER_VER}/g" all-in-one/configs/heapster.yaml.conf \
> all-in-one/configs/heapster.yaml

sed "s/k8s.gcr.io/${PRIVATE_REGISTRY}/g; s/v4.4.3/${K8S_HEAPSTER_GRAFANA_VER}/g" all-in-one/configs/grafana.yaml.conf \
> all-in-one/configs/grafana.yaml
#Depend on your request
#you may change GF_SERVER_ROOT_URL to /api/v1/namespaces/kube-system/services/monitoring-grafana/proxy
#instead of /


sed "s/k8s.gcr.io/${PRIVATE_REGISTRY}/g; s/v1.3.3/${K8S_HEAPSTER_INFLUXDB_VER}/g" all-in-one/configs/influxdb.yaml.conf \
> all-in-one/configs/influxdb.yaml

for conf in heapster.yaml grafana.yaml influxdb.yaml heapster-rbac.yaml ; do
  scp all-in-one/configs/$conf ${KUBE_MASTER_SERV_01}:/etc/kubernetes/addons
done
```

## 2. Start service

```bash
for conf in heapster-rbac.yaml heapster.yaml grafana.yaml influxdb.yaml ; do
  ssh ${KUBE_MASTER_SERV_01} kubectl apply -f /etc/kubernetes/addons/$conf
done
```
Visit the grafana web interface at `https://${KUBE_APISERVER}/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy` which is shown by `ssh ${KUBE_MASTER_SERV_01} kubectl cluster-info`

Rreference:
https://github.com/kubernetes/heapster/blob/master/docs/influxdb.md
