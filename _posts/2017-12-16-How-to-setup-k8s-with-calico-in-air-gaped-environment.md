---
layout: post
title: "Deploy K8s + Calico network in air-gapped environment"
date: 2017-12-18 14:16:01 +0800
tags: ["Kubenetes","Calico"]
---

## Pre-defined environment

- Internet availabe host: **localhost** , the host have to be with the same architecture as air-gaped host ,like amd_64, x86_64
- Air-gapped host:
  - **master01.airgapped.org**, 192.168.122.163
  - **node01.airgapped.org**, 192.168.122.*
  - **node02.airgapped.org**, 192.168.122.*
  - **node03.airgapped.org**, 192.168.122.*
- K8s master runs on **master01.airgaped.org**
- K8s `$CLUSTER_IP` 172.16.0.1/16,`$POD_IP` 172.18.0.1/16
- Private registry service runs on **registry.airgaped.org** (hosted on **master01.airgaped.org**) with port 443 and TLS enabled. See [Deploy a private docker registry in air-gaped environment](https://highland0971.github.io/2017/12/18/Deploy-a-private-docker-registry-in-air-gaped-environment.html)


The following process is composed under the instruction of [Creating a Custom Cluster from Scratch](https://kubernetes.io/docs/getting-started-guides/scratch/#downloading-and-extracting-kubernetes-binaries) on Kubernetes.io. You may refer to orginal documents to get detailed information.


## 1. Download K8s Packages

 As described in [Downloading and Extracting Kubernetes Binaries](https://kubernetes.io/docs/getting-started-guides/scratch/#downloading-and-extracting-kubernetes-binaries), you have to download the latest kubernetes bootstrap binaries from [Github](https://github.com/kubernetes/kubernetes/releases), on **localhost** ：

```bash
curl -L -o kubernetes-v1_9_0.tar.gz \ https://github.com/kubernetes/kubernetes/releases/download/v1.9.0/kubernetes.tar.gz
tar -zxf kubernetes-v1_9_0.tar.gz
pushd kubernetes/cluster
./get-kube-binaries.sh
```
You may have to try multiple times as poor Internet connection from CN to dl.k8s.io.

As result, `kubernetes-server-linux-amd64.tar.gz`  and `kubernetes-client-linux-amd64.tar.gz`  were downloaded into `../kubernetes/server` and `/kubernetes/client`. However the `kubernetes-client-linux-amd64.tar.gz` has been extarted to `kubernetes/platforms/linux/amd64`
You shall extract `kubernetes-server-linux-amd64.tar.gz`,`kubernetes-salt.tar.gz` by yourself, move the extracted files to proper directory amd make new archive.
```bash
popd
pushd kubernetes/server
tar -zxf kubernetes-server-linux-amd64.tar.gz -C ../../
tar -zxf kubernetes-salt.tar.gz -C ../../
popd
tar -czf kubernetes-amd64-downloaded.tar.gz
scp kubernetes-amd64-downloaded.tar.gz user@airgaped.org
```

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
sudo mv kubernetes $K8S_BASE_PATH
sudo chown +R `id -un`:`id -gn` $K8S_BASE_PATH/kubernetes
```
You can replace `$K8S_BASE_PATH` with `/srv` or whatever path suits your environment.

## 2. Master and Node Environment repare
- Update iptables rules and end forward
  ```bash
  cat > /tmp/k8s.conf <<EOF
  net.ipv4.ip_forward = 1
  net.bridge.bridge-nf-call-ip6tables = 1
  net.bridge.bridge-nf-call-iptables = 1
  EOF
  sudo mv /tmp/k8s.conf /etc/sysctl.d/
  sudo sysctl -p /etc/sysctl.d/k8s.conf
  ```
- Set binary search PATH

  **For non-privillaged user**

    ```bash
    sudo sh -c "echo 'export PATH=$PATH:/srv/kubernetes/client/bin' > /etc/profile.d/k8s.sh"
    sudo source /etc/profile.d/k8s.sh
    ```
  **For privilaged user**

    Append `/srv/kubernetes/client/bin` to sudoer file's `secure_path`, like
    >Defaults    secure_path = /sbin:/bin:/usr/sbin:/usr/bin:/srv/kubernetes/client/bin

    With command `sudo visudo`.

- Disable Swap file

  By disable **swap** entry in `/etc/fstab` and run
  ```
  sudo swapoff -aV
  ```

- Disable SeLinux

  SELINUX must be disabled, otherwise the docker contianer can not access host path file , encounter problems like level=fatal msg="open /cert/ca.crt: permission denied".

  ```bash
  sudo setenforce 0
   sed 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/selinux/config | sudo tee /etc/selinux/config
  ```

  Verify the whether selinux is disabled:

  `$ getenforce`
  >Permissive


## 4. Perpare certificates for K8s
Download certificates tool **cfssl** on  **localhost** and copy to **master01.airgapped.org**

On **localhost**
```bash
curl -L -o cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
curl -L -o cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
curl -L -o cfssl-certinfo https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl*
tar -czf cfssl_tool.tar.gz cfssl*
scp cfssl_tool.tar.gz master01.airgapped.org:$K8S_BASE_PATH/kubernetes
```

On **master01.airgapped.org**

- Create CA config file
  ```bash
  export K8S_BASE_PATH=/srv
  cd $K8S_BASE_PATH/kubernetes
  tar -zxf cfssl_tool.tar.gz
  mkdir certs

  cat > $K8S_BASE_PATH/kubernetes/certs/ca-config.json <<EOF
  {
    "signing": {
      "default": {
        "expiry": "87600h"
      },
      "profiles": {
        "kubernetes": {
          "usages": [
              "signing",
              "key encipherment",
              "server auth",
              "client auth"
          ],
          "expiry": "87600h"
        }
      }
    }
  }
  EOF
  ```

- Create CA [CSR](http://www.sohu.com/a/133008588_604699) config file
  ```bash
  export K8S_BASE_PATH=/srv
  cd $K8S_BASE_PATH/kubernetes
  cat > $K8S_BASE_PATH/kubernetes/certs/ca-csr.json <<EOF
  {
    "CN": "kubernetes",
    "key": {
      "algo": "rsa",
      "size": 2048
    },
    "names": [
      {
        "C": "CN",
        "ST": "DreamState",
        "L": "DreamCity",
        "O": "k8s",
        "OU": "System"
      }
    ]
  }
  EOF
  ```
- Generate CA key (`ca-key.pem`) and certificate (`ca.pem`):
  ```bash
  export K8S_BASE_PATH=/srv
  cd $K8S_BASE_PATH/kubernetes
  ./cfssl gencert -initca certs/ca-csr.json | ./cfssljson -bare certs/ca
  sudo cp certs/ca*.pem /etc/kubernetes/certs/
  ```
  `$ ls certs`
  >ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem

- Generate ETCD Server key (`etcd-server-key.pem`) and certificate (`etcd-server.pem`)

  `etcd-server*.pem` pair is used to client auth etcd server with trusted ca-cert
  ```bash
  export K8S_BASE_PATH=/srv
  export MASTER_IP="192.168.122.163"
  export ETCD_SERV01_IP=${MASTER_IP}
  cd $K8S_BASE_PATH/kubernetes
  cat > $K8S_BASE_PATH/kubernetes/certs/etcd-server-csr.json <<EOF
  {
      "CN": "etcd-cluster",
      "hosts": [
        "$ETCD_SERV01_IP",
        "$ETCD_SERV02_IP",
        "$ETCD_SERV03_IP",
        "master01.airgaped.org",
        "etcd01.airgaped.org",
        "etcd02.airgaped.org",
        "etcd03.airgaped.org"
      ],
      "key": {
          "algo": "rsa",
          "size": 2048
      },
      "names": [
      ]
  }
  EOF
  ./cfssl gencert -ca=certs/ca.pem -ca-key=certs/ca-key.pem \
    -config=certs/ca-config.json -profile=kubernetes \
    certs/etcd-server-csr.json | \
    ./cfssljson -bare certs/etcd-server
  sudo cp certs/etcd-server*.pem /etc/kubernetes/certs
  ```

- Generate API Server key (`apiserver-key.pem`)and certificate(`apiserver.pem`)

  ```bash
  export K8S_BASE_PATH=/srv
  export MASTER_IP="192.168.122.163"
  export CLUSTER_IP="172.16.0.1"
  export KUBE_APISERVER="https://${MASTER_IP}:6443"
  cd $K8S_BASE_PATH/kubernetes
  cat > $K8S_BASE_PATH/kubernetes/certs/apiserver-csr.json <<EOF
  {
      "CN": "kube-apiserver",
      "hosts": [
      "${MASTER_IP}",
      "127.0.0.1",
      "${CLUSTER_IP}",
      "master01.airgaped.org",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster.local"
      ],
      "key": {
          "algo": "rsa",
          "size": 2048
      },
      "names": [
          {
          }
      ]
  }
  EOF
  ./cfssl gencert -ca=certs/ca.pem -ca-key=certs/ca-key.pem \
    -config=certs/ca-config.json -profile=kubernetes \
    certs/apiserver-csr.json | \
    ./cfssljson -bare certs/apiserver
  sudo cp certs/apiserver*.pem /etc/kubernetes/certs
  ```

- Generate API Server kubelet client key (`apiserver-kubelet-client-key.pem`)and certificate(`apiserver-kubelet-client.pem`)

  ```bash
  export K8S_BASE_PATH=/srv
  cd $K8S_BASE_PATH/kubernetes
  cat > $K8S_BASE_PATH/kubernetes/certs/apiserver-kubelet-client-csr.json <<EOF
  {
      "CN": "kube-apiserver-kubelet-client",
      "hosts": [
      ],
      "key": {
          "algo": "rsa",
          "size": 2048
      },
      "names": [
          {
            "O": "system:masters"
          }
      ]
  }
  EOF
  ./cfssl gencert -ca=certs/ca.pem -ca-key=certs/ca-key.pem \
    -config=certs/ca-config.json -profile=kubernetes \
    certs/apiserver-kubelet-client-csr.json | \
    ./cfssljson -bare certs/apiserver-kubelet-client
  sudo cp certs/apiserver-kubelet-client*.pem /etc/kubernetes/certs
  ```

- Generate kube-controller-manager key (`kube-controller-manager-key.pem`)and certificate(`kube-controller-manager.pem`)

  ```bash
  export K8S_BASE_PATH=/srv
  export MASTER_IP="192.168.122.163"
  export KUBE_APISERVER="https://${MASTER_IP}:6443"
  cd $K8S_BASE_PATH/kubernetes
  cat > $K8S_BASE_PATH/kubernetes/certs/kube-controller-manager-csr.json <<EOF
  {
      "CN": "kube-controller-manager",
      "hosts": [
      ],
      "key": {
          "algo": "rsa",
          "size": 2048
      },
      "names": [
          {
            "O": "system:masters"
          }
      ]
  }
  EOF
  ./cfssl gencert -ca=certs/ca.pem -ca-key=certs/ca-key.pem \
    -config=certs/ca-config.json -profile=kubernetes \
    certs/kube-controller-manager-csr.json | \
    ./cfssljson -bare certs/kube-controller-manager
  sudo cp certs/kube-controller-manager*.pem /etc/kubernetes/certs
  ```    
- Generate API Server etcd client key (`apiserver-etcd-client-key.pem`) and certificate (`apiserver-etcd-client.pem`)

  ```bash
  export K8S_BASE_PATH=/srv
  cd $K8S_BASE_PATH/kubernetes
  cat > $K8S_BASE_PATH/kubernetes/certs/apiserver-etcd-client-csr.json <<EOF
  {
      "CN": "kube-apiserver-etcd-client",
      "hosts": [
      ],
      "key": {
          "algo": "rsa",
          "size": 2048
      },
      "names": [
          {
            "O": "system:etcd-clients"
          }
      ]
  }
  EOF
  ./cfssl gencert -ca=certs/ca.pem -ca-key=certs/ca-key.pem \
    -config=certs/ca-config.json -profile=kubernetes \
    certs/apiserver-etcd-client-csr.json | \
    ./cfssljson -bare certs/apiserver-etcd-client
  sudo cp certs/apiserver-etcd-client*.pem /etc/kubernetes/certs
  ```
- Generate Service Account Private Key(`sa.key`) and Public Key(`sa.pub`)
  ```bash
  export K8S_BASE_PATH=/srv
  cd $K8S_BASE_PATH/kubernetes
  sudo openssl genrsa -out certs/sa.key 2048
  sudo openssl rsa -in certs/sa.key -pubout -out certs/sa.pub
  ```

- Generate admin  key (`admin-key.pem`) and certificate (`admin.pem`).

  By set the user group to **system:masters** in "O" field,to authorize the admin user with full access to the k8s cluster. The k8s `cluster-admin` has pre-defined Group  **system:masters** to Role **cluster-admin** bound.
  ```bash
  cd $K8S_BASE_PATH/kubernetes
  cat > certs/admin-csr.json << EOF
  {
    "CN": "admin",
    "hosts": [],
    "key": {
      "algo": "rsa",
      "size": 2048
    },
    "names": [
      {
        "C": "CN",
        "ST": "DreamState",
        "L": "DreamCity",
        "O": "system:masters",
        "OU": "System"
      }
    ]
  }
  EOF
  ./cfssl gencert -ca=certs/ca.pem -ca-key=certs/ca-key.pem -config=certs/ca-config.json -profile=kubernetes certs/admin-csr.json | ./cfssljson -bare certs/admin
  sudo cp certs/admin*.pem /etc/kubernetes/certs
  ```

  `$ ls certs/admin*`
  >certs/admin.csr  certs/admin-csr.json  certs/admin-key.pem  certs/admin.pem


- Generate kube-proxy  key (`kube-proxy-key.pem`) and certificate (`kube-proxy.pem`).

  By set the CN to **system:kube-proxy** ,to set user **system:kube-proxy** bind with Role **system:node-proxier**.

  ```bash
  cd $K8S_BASE_PATH/kubernetes
  cat > certs/kube-proxy-csr.json << EOF
  {
    "CN": "system:kube-proxy",
    "hosts": [],
    "key": {
      "algo": "rsa",
      "size": 2048
    },
    "names": [
      {
        "C": "CN",
        "ST": "DreamState",
        "L": "DreamCity",
        "O": "k8s",
        "OU": "System"
      }
    ]
  }
  EOF

  ./cfssl gencert -ca=certs/ca.pem -ca-key=certs/ca-key.pem -config=certs/ca-config.json -profile=kubernetes certs/kube-proxy-csr.json | ./cfssljson -bare certs/kube-proxy
  ```

  `$ ls certs/kube-proxy*`
  >certs/kube-proxy.csr  certs/kube-proxy-csr.json  certs/kube-proxy-key.pem  certs/kube-proxy.pem

- Verify generated key file with
  ```bash
  openssl x509  -noout -text -in your-key-file.pem

  #OR

  cfssl-certinfo -cert your-key-file.pem
  ```
- Copy key files to /etc/kubernates/certs
  ```bash
  sudo mkdir /etc/kubernetes/certs -p
  sudo cp $K8S_BASE_PATH/kubernetes/certs/*.pem /etc/kubernetes/certs
  sudo chmod 644 /etc/kubernetes/certs/*
  ```
  `$ ls /etc/kubernetes/certs/`
  >admin-key.pem  admin.pem  ca-key.pem  ca.pem  kube-proxy-key.pem  kube-proxy.pem  kubernetes-key.pem  kubernetes.pem

Refer to:
- [创建TLS证书和秘钥](https://jimmysong.io/kubernetes-handbook/practice/create-tls-and-secret-key.html)
- [Certificates](https://kubernetes.io/docs/concepts/cluster-administration/certificates/)

  #### Distributing Self-Signed CA Certificate
  **TODO need a without test**
  >$ sudo cp ca.crt /usr/local/share/ca-certificates/kubernetes.crt
  $ sudo update-ca-certificates
  Updating certificates in /etc/ssl/certs...
  1 added, 0 removed; done.
  Running hooks in /etc/ca-certificates/update.d....
  done.
  >

## 5.Setup kubeconfig files
- **For master kubelet**
  ```bash
  export K8S_BASE_PATH=/srv
  export MASTER_IP="192.168.122.163"

  cd /etc/kubernetes

  sudo kubectl config set-cluster kubernetes \
   --certificate-authority=/etc/kubernetes/certs/ca.pem \
   --embed-certs=true \
   --server=https://${MASTER_IP}:6443 \
   --kubeconfig=./kubelet.conf

  sudo kubectl config set-credentials system:node:master01 \
   --client-certificate=./certs/apiserver-kubelet-client.pem \
   --client-key=./certs/apiserver-kubelet-client-key.pem \
   --embed-certs=true \
   --kubeconfig=./kubelet.conf

   sudo kubectl config set-context default \
   --cluster=kubernetes \
   --user=system:node:master01 \
   --kubeconfig=./kubelet.conf

   sudo kubectl config use-context default --kubeconfig=./kubelet.conf

   sudo chmod 640 *.conf
  ```
- **For kube-controller-manager**
  ```bash
  export K8S_BASE_PATH=/srv
  export MASTER_IP="192.168.122.163"

  cd /etc/kubernetes

  sudo kubectl config set-cluster kubernetes \
   --certificate-authority=/etc/kubernetes/certs/ca.pem \
   --embed-certs=true \
   --server=https://${MASTER_IP}:6443 \
   --kubeconfig=./controller-manager.conf

  sudo kubectl config set-credentials system:kube-controller-manager \
   --client-certificate=./certs/kube-controller-manager.pem \
   --client-key=./certs/kube-controller-manager-key.pem \
   --embed-certs=true \
   --kubeconfig=./controller-manager.conf

   sudo kubectl config set-context default \
   --cluster=kubernetes \
   --user=system:kube-controller-manager \
   --kubeconfig=./controller-manager.conf

   sudo kubectl config use-context default --kubeconfig=./controller-manager.conf
   sudo chmod 640 *.conf
  ```

- **For node bootstrap**
  ```bash
  export MASTER_IP="192.168.122.163"
  export KUBE_APISERVER="https://${MASTER_IP}:6443"
  cd /etc/kubernetes
  export BOOTSTRAP_TOKEN=$(dd if=/dev/urandom bs=128 count=1 2>/dev/null | base64 | tr -d "=+/" | dd bs=32 count=1 2>/dev/null)
  sudo sh -c "echo ${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,system:kubelet-bootstrap > /etc/kubernetes/known_tokens.csv"

  sudo kubectl config set-cluster kubernetes \
   --certificate-authority=/etc/kubernetes/certs/ca.pem \
   --embed-certs=true \
   --server=${KUBE_APISERVER} \
   --kubeconfig=./bootstrap.conf

  sudo kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=./bootstrap.conf

  sudo kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=./bootstrap.conf

  sudo kubectl config use-context default --kubeconfig=./bootstrap.conf

  sudo chmod 640 *.conf
  ```

- **For Kube-Proxy**
  ```bash
  export KUBE_APISERVER="https://192.168.122.163:6443"
  cd /etc/kubernetes
  sudo kubectl config set-cluster kubernetes \
   --certificate-authority=./certs/ca.pem \
   --embed-certs=true \
   --server=${KUBE_APISERVER} \
   --kubeconfig=./kube-proxy.conf

  sudo kubectl config set-credentials kube-proxy \
  --client-certificate=./certs/kube-proxy.pem \
  --client-key=./certs/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=./kube-proxy.conf

  sudo kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=./kube-proxy.conf

  sudo kubectl config use-context default \
  --kubeconfig=/etc/kubernetes/kube-proxy.conf

  sudo chmod 644 *.conf
  ```

- **For administrator**
  ```bash
  export KUBE_APISERVER="https://192.168.122.163:6443"
  cd /etc/kubernetes
  sudo kubectl config set-cluster kubernetes \
   --certificate-authority=./certs/ca.pem \
   --embed-certs=true \
   --server=${KUBE_APISERVER} \
   --kubeconfig=./admin.conf

  sudo kubectl config set-credentials admin \
  --client-certificate=./certs/admin.pem \
  --client-key=./certs/admin-key.pem \
  --embed-certs=true \
  --kubeconfig=./admin.conf

  sudo kubectl config set-context default \
  --cluster=kubernetes \
  --user=admin \
  --kubeconfig=./admin.conf

  sudo kubectl config use-context default --kubeconfig=/etc/kubernetes/admin.conf
  sudo chmod 644 *.conf
  ```

## 6.Setup kubelet service
```bash
export KUBE_APISERVER="https://192.168.122.163:6443"
export K8S_BASE_PATH=/srv
export KUBE_ETC=/etc/kubernetes

cat > /tmp/kubelet.service << EOF
[Unit]
Description=Kubernetes kubelet deamon
Documentation=https://jimmysong.io/kubernetes-handbook/practice/master-installation.html
After=network.target

[Service]
ExecStart=${K8S_BASE_PATH}/kubernetes/server/bin/kubelet \
        --pod-manifest-path=${KUBE_ETC}/manifests \
        --allow-privileged=true \
        --kubeconfig=${KUBE_ETC}/kubelet.conf \
        --bootstrap-kubeconfig=${KUBE_ETC}/bootstrap-kubelet.conf \
        --client-ca-file=${KUBE_ETC}/certs/ca.pem \
        --cert-dir=/var/lib/kubelet/pki \
        --cluster-dns=172.16.0.10 \
        --cluster-domain=cluster.local \
        --cgroup-driver=systemd \

Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
sudo mv /tmp/kubelet.service /usr/lib/systemd/system/
sudo systemctl enable kubelet && sudo systemctl start kubelet
```

##7. Setup etcd deamon pod
On **localhost**
```bash
docker pull quay.io/coreos/etcd:v3.1.10
docker pull lhcalibur/pause-amd64:3.0
docker save quay.io/coreos/etcd:v3.1.10 -o tmp_etcd_v3_1_10.tar
docker save docker.io/lhcalibur/pause-amd64:3.0 -o tmp_pause_amd64_3_0.tar
scp tmp_*.tar user@master01.airgapped.org:~/
```
On **master01.airgapped.org**
```bash
docker load -i tmp_etcd_v3_1_0.tar
docker load -i tmp_pause_amd64_3_0.tar
docker tag quay.io/coreos/etcd:v3.1.10 registry.airgaped.org/etcd:v3.1.10
docker tag docker.io/lhcalibur/pause-amd64:3.0 registry.airgaped.org/pause-amd64:3.0
docker tag docker.io/lhcalibur/pause-amd64:3.0 gcr.io/google_containers/pause-amd64:3.0
docker push registry.airgaped.org/pause-amd64:3.0
docker push registry.airgaped.org/etcd:v3.1.10

sudo mkdir -p /etc/kubernetes/manifests
sudo mkdir -p /mnt/master-pd/var/etcd

export MASTER_IP="192.168.122.163"
export ETCD_SERV01_IP=${MASTER_IP}

cat >/tmp/etcd.manifest << EOF
{
"apiVersion": "v1",
"kind": "Pod",
"metadata": {
  "name":"k8s-etcd-01",
  "namespace": "kube-system",
  "annotations": {
    "scheduler.alpha.kubernetes.io/critical-pod": ""
  }
},
"spec":{
"hostNetwork": true,
"containers":[
    {
    "name": "k8s-etcd-01",
    "image": "registry.airgaped.org/etcd:v3.1.10",
    "resources": {
      "requests": {
        "cpu": "100m"
      }
    },
    "command": [
        "/bin/sh",
        "-c",
        "/usr/local/bin/etcd 1>&2"
            ],
    "env": [
      { "name": "ETCD_NAME","value": "etcd-01"},
      { "name": "ETCD_LISTEN_PEER_URLS","value": "https://${ETCD_SERV01_IP}:2380"},
      { "name": "ETCD_INITIAL_ADVERTISE_PEER_URLS","value": "https://${ETCD_SERV01_IP}:2380"},
      { "name": "ETCD_ADVERTISE_CLIENT_URLS","value": "https://${ETCD_SERV01_IP}:2379"},
      { "name": "ETCD_LISTEN_CLIENT_URLS","value": "https://${ETCD_SERV01_IP}:2379,https://127.0.0.1:2379"},
      { "name": "ETCD_DATA_DIR","value": "/var/etcd/data-01"},
      { "name": "ETCD_INITIAL_CLUSTER_STATE","value": "new"},
      { "name": "ETCD_INITIAL_CLUSTER","value": "etcd-01=https://${ETCD_SERV01_IP}:2380"},
      { "name": "ETCD_PEER_CERT_FILE","value": "/etc/kubernetes/certs/etcd-server.pem"},
      { "name": "ETCD_PEER_KEY_FILE","value": "/etc/kubernetes/certs/etcd-server-key.pem"},
      { "name": "ETCD_CERT_FILE","value": "/etc/kubernetes/certs/etcd-server.pem"},
      { "name": "ETCD_KEY_FILE","value": "/etc/kubernetes/certs/etcd-server-key.pem"},
      { "name": "ETCD_PEER_CLIENT_CERT_AUTH","value": "true"},
      { "name": "ETCD_PEER_TRUSTED_CA_FILE","value": "/etc/kubernetes/certs/ca.pem"},
      { "name": "ETCD_CLIENT_CERT_AUTH","value": "true"},
      { "name": "ETCD_TRUSTED_CA_FILE","value": "/etc/kubernetes/certs/ca.pem"}
        ],
    "livenessProbe": {
      "httpGet": {
        "scheme": "HTTP",
        "host": "127.0.0.1",
        "port": 2379,
        "path": "/health"
      },
      "initialDelaySeconds": 15,
      "timeoutSeconds": 15
    },
    "ports": [
      { "name": "serverport",
        "containerPort": 2380,
        "hostPort": 2380
      },
      { "name": "clientport",
        "containerPort": 2379,
        "hostPort": 2379
      }
        ],
    "volumeMounts": [
      { "name": "varetcd",
        "mountPath": "/var/etcd",
        "readOnly": false
      },
      { "name": "varlogetcd",
        "mountPath": "/var/log/etcd-01.log",
        "readOnly": false
      },
      { "name": "etc",
        "mountPath": "/etc/kubernetes",
        "readOnly": false
      }
    ]
    }
],
"volumes":[
  { "name": "varetcd",
    "hostPath": {
        "path": "/mnt/master-pd/var/etcd"}
  },
  { "name": "varlogetcd",
    "hostPath": {
        "path": "/var/log/etcd-01.log",
        "type": "FileOrCreate"}
  },
  { "name": "etc",
    "hostPath": {
        "path": "/etc/kubernetes"}
  }
]
}}
EOF
sudo mv /tmp/etcd.manifest /etc/kubernetes/manifests/
```
For full configuration,please refer [Configuration flags](https://coreos.com/etcd/docs/latest/op-guide/configuration.html).

Your can verify whether the etcd successfully started by `$cat /var/log/etcd-01.log` and
```bash
$ docker run -ti -v /etc/kubernetes/certs:/etc/certs \
 registry.airgaped.org/etcd:v3.1.10 \
/bin/sh -c "ETCDCTL_API=3 /usr/local/bin/etcdctl  \
--cacert=/etc/certs/ca.pem \
--cert=/etc/certs/apiserver-etcd-client.pem \
--key=/etc/certs/apiserver-etcd-client-key.pem \
--endpoints="https://192.168.122.163:2379" \
endpoint health"
```
`OUTPUT` like:
>2017-12-24 16:08:51.697064 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
https://192.168.122.163:2379 is healthy: successfully committed proposal: took = 1.310057ms


##8. Setup Api Server deamon pod

On **master01.airgapped.org**
```bash
export MASTER_IP="192.168.122.163"
export CLUSTER_IP="172.16.0.1"
export SERVICE_CLUSTER_IP_RANGE="${CLUSTER_IP}/16"
export RECOMMENDED_LIST="Initializers,NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota"
docker load -i /srv/kubernetes/server/bin/kube-apiserver.tar
docker tag gcr.io/google_containers/kube-apiserver:v1.9.0 registry.airgaped.org/kube-apiserver:v1.9.0
docker push registry.airgaped.org/kube-apiserver:v1.9.0
docker rmi gcr.io/google_containers/kube-apiserver:v1.9.0
sudo mkdir -p /var/log/kube-apiserver
cat >/tmp/kube-apiserver.manifest << EOF
{
  "kind": "Pod",
  "apiVersion": "v1",
  "metadata": {
    "name": "kube-apiserver",
    "namespace": "kube-system",
    "scheduler.alpha.kubernetes.io/critical-pod": "",
    "labels":{
      "component":"kube-apiserver",
      "tier":"control-plane"
    }
  },
  "spec": {
    "hostNetwork": true,
    "containers": [
      {
        "name": "kube-apiserver",
        "image": "registry.airgaped.org/kube-apiserver:v1.9.0",
        "command": [
          "/usr/local/bin/kube-apiserver",
          "--advertise-address=${MASTER_IP}",
          "--service-cluster-ip-range=${SERVICE_CLUSTER_IP_RANGE}",
          "--allow-privileged=true",

          "--admission-control=${RECOMMENDED_LIST}",

          "--enable-bootstrap-token-auth=true",
          "--token-auth-file=/etc/kubernetes/known_tokens.csv",

          "--client-ca-file=/etc/kubernetes/certs/ca.pem",
          "--service-account-key-file=/etc/kubernetes/certs/ca.pem",

          "--tls-cert-file=/etc/kubernetes/certs/apiserver.pem",
          "--tls-private-key-file=/etc/kubernetes/certs/apiserver-key.pem",

          "--kubelet-certificate-authority=/etc/kubernetes/certs/ca.pem",
          "--kubelet-client-certificate=/etc/kubernetes/certs/apiserver-kubelet-client.pem",
          "--kubelet-client-key=/etc/kubernetes/certs/apiserver-kubelet-client-key.pem",

          "--etcd-servers=https://${MASTER_IP}:2379",
          "--etcd-cafile=/etc/kubernetes/certs/ca.pem",
          "--etcd-certfile=/etc/kubernetes/certs/apiserver-etcd-client.pem",
          "--etcd-keyfile=/etc/kubernetes/certs/apiserver-etcd-client-key.pem",

          "--log-dir=/var/log",
          "--audit-log-path=/var/log/audit.log"
        ],
        "resources":{
          "requests":{
            "cpu":"250m"
          }
        },
        "ports": [
          {
            "name": "https",
            "hostPort": 6443,
            "containerPort": 6443
          }
        ],
        "volumeMounts": [
          {
            "name": "srvkube",
            "mountPath": "/etc/kubernetes",
            "readOnly": true
          },
          {
            "name":"logdir",
            "mountPath": "/var/log"
          }
        ],
        "livenessProbe": {
          "httpGet": {
            "scheme": "HTTPS",
            "host": "${MASTER_IP}",
            "port": 8080,
            "path": "/healthz"
          },
          "initialDelaySeconds": 15,
          "timeoutSeconds": 15,
          "failureThreshold":8
        }
      }
    ],
    "volumes": [
      {
        "name": "srvkube",
        "hostPath": {
          "path": "/etc/kubernetes",
          "type": "DirectoryOrCreate"
        }
      },
      {
        "name":"logdir",
        "hostPath": {
          "path": "/var/log/kube-apiserver",
          "type": "DirectoryOrCreate"
        }
      }
    ]
  }
}
EOF
sudo mv /tmp/kube-apiserver.manifest /etc/kubernetes/manifests/

sudo systemctl restart kubelet
```

##9. Setup kube-controller-manager deamon pod

On **master01.airgapped.org**
```bash
export MASTER_IP="192.168.122.163"
export CLUSTER_IP="172.16.0.1"
export SERVICE_CLUSTER_IP_RANGE="${CLUSTER_IP}/16"
export RECOMMENDED_LIST="Initializers,NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota"
docker load -i /srv/kubernetes/server/bin/kube-controller-manager.tar
docker tag gcr.io/google_containers/kube-controller-manager:v1.9.0 registry.airgaped.org/kube-controller-manager:v1.9.0
docker push registry.airgaped.org/kube-controller-manager:v1.9.0
docker rmi gcr.io/google_containers/kube-controller-managerr:v1.9.0
sudo mkdir -p /var/log/kube-controller-manager
cat >/tmp/kube-controller-manager.manifest << EOF
{
  "kind": "Pod",
  "apiVersion": "v1",
  "metadata": {
    "name": "kube-controller-manager",
    "namespace": "kube-system",
    "scheduler.alpha.kubernetes.io/critical-pod": "",
    "labels":{
      "component":"kube-controller-manager",
      "tier":"control-plane"
    }
  },
  "spec": {
    "hostNetwork": true,
    "containers": [
      {
        "name": "kube-controller-manager",
        "image": "registry.airgaped.org/kube-controller-manager:v1.9.0",
        "command": [
          "/usr/local/bin/kube-controller-manager",
          "--address=127.0.0.1",
          "--leader-elect=true",
          "--controllers=*,bootstrapsigner,tokencleaner",
          "--kubeconfig=/etc/kubernetes/controller-manager.conf",
          "--use-service-account-credentials=true",
          "--root-ca-file=/etc/kubernetes/certs/ca.pem",
          "--service-account-private-key-file=/etc/kubernetes/certs/sa.key",
          "--cluster-signing-key-file=/etc/kubernetes/certs/ca-key.pem",
          "--cluster-signing-cert-file=/etc/kubernetes/pki/ca.pem",

          "--log-dir=/var/log"
        ],
        "livenessProbe": {
          "httpGet": {
            "scheme": "HTTP",
            "host": "127.0.0.1",
            "port": 10252,
            "path": "/healthz"
          },
          "initialDelaySeconds": 15,
          "timeoutSeconds": 15,
          "failureThreshold":8
        },
        "resources":{
          "requests":{
            "cpu":"250m"
          }
        },
        "ports": [
          {
            "name": "http",
            "hostPort": 10252,
            "containerPort": 10252
          }
        ],
        "volumeMounts": [
          {
            "name": "srvkube",
            "mountPath": "/etc/kubernetes",
            "readOnly": true
          },
          {
            "name":"logdir",
            "mountPath": "/var/log"
          }
        ]
      }
    ],
    "volumes": [
      {
        "name": "srvkube",
        "hostPath": {
          "path": "/etc/kubernetes"
        }
      },
      {
        "name":"logdir",
        "hostPath": {
          "path": "/var/log/kube-controller-manager",
          "type": "DirectoryOrCreate"
        }
      },
      {
        "name":"flexvolume-dir",
        "hostPath": {
          "path": "/usr/libexec/kubernetes/kubelet-plugins/volume/exec",
          "type": "DirectoryOrCreate"
        }
      }
    ]
  }
}
EOF
sudo mv /tmp/kube-controller-manager.manifest /etc/kubernetes/manifests/
sudo systemctl restart kubelet
```


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
