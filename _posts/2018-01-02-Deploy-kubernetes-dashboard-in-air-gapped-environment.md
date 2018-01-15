---
layout: post
title: "Deploy kubernetes-dashboard in air-gapped environment"
date: 2018-01-02 10:48:01 +0800
tags: ["Kubenetes","Dashboard"]
---

This article describes kubernetes dashboard deployment in an air-gapped environments, based on the [pre-installed k8s cluster](https://highland0971.github.io/2017/12/29/Deploy-K8s-Calico-in-air-gapped-environment.html)

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
```

## 1. Prepare required images and yaml config
- Download kubernetes-dashboard container image and push to private registry
We omitted the docker save & load procedure from Internet available host to air-gapped cluster `INSTALLATION_BASE`, you can refer artical revelent section in [pre-installed k8s cluster](https://highland0971.github.io/2017/12/29/Deploy-K8s-Calico-in-air-gapped-environment.html).

```bash
#For kubernetes dashboard image
#On Internet available host
docker pull highland0971/${K8S_DASHBOARD_IMG_NAME}:${K8S_DASHBOARD_VER} && \
docker tag highland0971/${K8S_DASHBOARD_IMG_NAME}:${K8S_DASHBOARD_VER} ${PRIVATE_REGISTRY}/${K8S_DASHBOARD_IMG_NAME}:${K8S_DASHBOARD_VER} && \
docker save ${PRIVATE_REGISTRY}/${K8S_DASHBOARD_IMG_NAME}:${K8S_DASHBOARD_VER} -o all-in-one/images/${K8S_DASHBOARD_IMG_NAME}:${K8S_DASHBOARD_VER}.tar

#On ${INSTALLATION_BASE}
ssh ${PRIVATE_REGISTRY_SERV} mkdir ~/images
scp all-in-one/images/${K8S_DASHBOARD_IMG_NAME}:${K8S_DASHBOARD_VER}.tar ${PRIVATE_REGISTRY_SERV}:~/images
ssh ${PRIVATE_REGISTRY_SERV} docker load -i ~/images/${K8S_DASHBOARD_IMG_NAME}:${K8S_DASHBOARD_VER}.tar
ssh ${PRIVATE_REGISTRY_SERV} docker push ${PRIVATE_REGISTRY}/${K8S_DASHBOARD_IMG_NAME}:${K8S_DASHBOARD_VER}
```

- Curl yaml from kubernetes-dashboard offical webset and modify according to your private registry location

```bash
curl -L "https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml" -o all-in-one/configs/kubernetes-dashboard.yaml.conf

sed "s/k8s.gcr.io/${PRIVATE_REGISTRY}/g" all-in-one/configs/kubernetes-dashboard.yaml.conf \
> all-in-one/configs/kubernetes-dashboard.yaml

scp all-in-one/configs/kubernetes-dashboard.yaml ${KUBE_MASTER_SERV_01}:/etc/kubernetes/addons
```
## 2. Start service

```bash
ssh ${KUBE_MASTER_SERV_01} kubectl apply -f /etc/kubernetes/addons/kubernetes-dashboard.yaml
#Check STATUS
ssh ${KUBE_MASTER_SERV_01} kubectl get all -n kube-system -l k8s-app=kubernetes-dashboard
# NAME                          DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
# deploy/kubernetes-dashboard   1         1         1            1           5m
#
# NAME                                 DESIRED   CURRENT   READY     AGE
# rs/kubernetes-dashboard-84f5d797bd   1         1         1         5m
#
# NAME                          DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
# deploy/kubernetes-dashboard   1         1         1            1           5m
#
# NAME                                 DESIRED   CURRENT   READY     AGE
# rs/kubernetes-dashboard-84f5d797bd   1         1         1         5m
#
# NAME                                       READY     STATUS    RESTARTS   AGE
# po/kubernetes-dashboard-84f5d797bd-lbts2   1/1       Running   0          5m
#
# NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
# svc/kubernetes-dashboard   ClusterIP   172.16.76.222   <none>        443/TCP   5m
```

## 3. Access the web UI and
You can access the dashboard ui via [https://<${KUBE_MASTER_SERV_01}>:<${KUBE_APISERVER_PORT}>/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/](https://<${KUBE_MASTER_SERV_01}>:<${KUBE_APISERVER_PORT}>/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/).
**Note** The shortcut described in [Accessing the Dashboard UI](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/) is deprecated.

The first time you access the URL you will encounter a failure response from browser，like this:

```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/ui\"",
  "reason": "Forbidden",
  "details": {

  },
  "code": 403
}
```
This is because the apiserver requires TLS client authentication, which is issued by `--client-ca-file` CA file. To obtain that certificate you can convert the `kube-admin` certificate file genenrated [previously](https://highland0971.github.io/2017/12/29/Deploy-K8s-Calico-in-air-gapped-environment.html) in k8s cluster setup to **PKCS12** format:

```bash
openssl pkcs12 -export -in certs/k8s/kube-admin.pem  -out certs/k8s/kube-admin.p12 -inkey certs/k8s/kube-admin-key.pem
# Enter Export Password:
# Verifying - Enter Export Password:
```
You have to secure the password which will be used when you import into Windows or other operation system. Then you have to copy the *.p12 file to your operation system and install it(Normally, by double-click the *.p12 file).
After successfully install, restart the browser and visit the URL again, you will be promoted to choose the certificate we have just installed.

**Note**: In the production environment, you have to replace `kube-admin` certificate with actual user certificate.

## 4. Supply the correct token to login
When you successfully opened the Login View, you will be promoted to supply kubeconfig file or token to login, simplely choose Token. The reason why and how to pick/create token is described [here](https://github.com/kubernetes/dashboard/issues/2474) and [here](https://github.com/kubernetes/dashboard/wiki/Access-control) and [here](https://github.com/kubernetes/dashboard/wiki/Creating-sample-user).

However, you may try to issue a new user with token and kubeconfig, describd in [Preparing Credentials](https://kubernetes.io/docs/getting-started-guides/scratch/#preparing-certs).

**References**
- [Installation](https://github.com/kubernetes/dashboard/wiki/Installation)
- [Accessing Dashboard 1.7.X and above](https://github.com/kubernetes/dashboard/wiki/Accessing-Dashboard---1.7.X-and-above)
- [Creating sample user](https://github.com/kubernetes/dashboard/wiki/Creating-sample-user)
- [Preparing Credentials](https://kubernetes.io/docs/getting-started-guides/scratch/#preparing-certs)
