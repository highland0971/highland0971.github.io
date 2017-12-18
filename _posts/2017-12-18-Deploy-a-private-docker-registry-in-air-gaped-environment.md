---
layout: post
title: "Deploy a private docker registry in air-gaped environment"
date: 2017-12-18 14:52:01 +0800
tags: ["Docker","Registry"]
---



## Pre-defined environment
- Internet availabe host: **localhost**
- Air-gaped host: **master01.airgaped.org**


# Prepare required Docker environments

## Pre-request

- You have to use the same linux distribution with your air-gaped host, which is  **master01.airgaped.org**, in this artical we adopt **CentOS 7** as host system.


- You'd have to obtain a **PURE** linux environment which **HAVE** Internet access without any post-installed packages as was installed in **master01.airgaped.org**.  Otherwise you may encounter failed dependencies for docker in the **master01.airgaped.org**.
- We assume that you have a **PURE** Internet access availabe host named **localhost**.

### 1. Download and install the required packages for Docker 
- [ ] On the **localhost** , run following commands:

```bash
mkdir ~/rpms
sudo yum update && yum install -y --downloadonly  --downloaddir=${HOME}/rpms docker
ssh user@master01.airgaped.org mkdir packages
#You can verify the outcome by type
ssh user@master01.airgaped.org "pwd && ls"
scp ${HOME}/rpms/* user@master01.airgaped.org:~/packages
```

- [ ] On the **master01.airgaped.org** , run following commands to install Docker environment

```bash
cd ~/packages
sudo rpm -ivh *
```

​	If you encoutner missing depandency error like
> warning: audit-libs-python-2.7.6-3.el7.x86_64.rpm: Header V3 RSA/SHA256 Signature, key ID f4a80eb5: NOKEY
> error: Failed dependencies:
>        libyajl.so.2()(64bit) is needed by oci-systemd-hook-1:0.1.14-1.git1ba44c6.el7.x86_64
>        libyajl.so.2()(64bit) is needed by oci-umount-2:2.3.0-1.git51e7c50.el7.x86_64
>        libcgroup is needed by policycoreutils-python-2.5-17.1.el7.x86_64

​	On the **localhost** you can run

```bash
sudo yum update && yum reinstall -y --downloadonly  --downloaddir=${HOME}/rpms missing-packages(like yajl libcgroup)
```

​	And redo the scp command for the missing rpms.

### 2. Setup Docker environment and Shoot!

- [ ] On the **master01.airgaped.org** , following the procedure defined in [Manage Docker as a non-root user](https://docs.docker.com/engine/installation/linux/linux-postinstall/#manage-docker-as-a-non-root-user) to add docker user group. 

      ```bash
      sudo groupadd docker
      sudo usermod -aG docker $USER
      ```
      
      You have to re-login to make current $USER group affect.

- [ ] On the **master01.airgaped.org** , disable SeLinux in order to allow docker container access the host path file.

      ```bash
      sudo setenforce 0
      sudo sed 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/selinux/config
      ```

- [ ] On the **master01.airgaped.org** , enable and start the docker deamon.

      ```bash
      sudo systemctl enable docker && sudo systemctl start docker
      ```

- [ ] On the **master01.airgaped.org** , check whether the docker's cgroupdriver is set to systemd

      `--exec-opt native.cgroupdriver=systemd`

      ```bash
      cat  /usr/lib/systemd/system/docker.service
      ```

      If not:

      - Edit the `/usr/lib/systemd/system/docker.service` file and add `--exec-opt native.cgroupdriver=systemd` to `/usr/lib/systemd/system/docker.service`'s ExecStart strings.

      - Restart the docker deamon service with 

      - ```bash 
        sudo systemctl reload-deamon docker && sudo systemctl restart docker
        ```


- [ ] On the **master01.airgaped.org** , check whether docker start successfully.

      ```bash
      systemctl status docker
      ```

# Setup private registry service

## Pre-request

- You have pulled the newset docker registry image from docker.io and export it to the tar archive.

 ```bash
   docker pull registry \
   && docker save docker.io/registry:latest -o docker-io.registry.tar
 ```
- You have to import the registry image to the **master01.airgaped.org** docker image repo cache

  On **master01.airgaped.org**

 ```bash
  docker load -i docker-io.registry.tar
 ```
### 1. Create HTTPS certificates required for docker private registry 

This procedure follows the example on [Openssl certificate creation on k8s official document](https://kubernetes.io/docs/concepts/cluster-administration/certificates/#openssl)

On **master01.airgaped.org** ：

- [ ] Prepare cert storage directory

     ```bash
     sudo mkdir /usr/lib/certs -p
     pushd /usr/lib/certs
     ```

- [ ] Generate CA key file

     ```bash
     openssl genrsa -out ca.key 2048
     ```

- [ ] Generate CA certificate file

      ```bash
      openssl req -x509 -new -nodes -key ca.key -subj "/CN=${MASTER_IP}" -days 10000 -out ca.crt
      ```

      You can replace `${MASTER_IP}` with your actual or persuade CA server ip address  like

      ```bash
      openssl req -x509 -new -nodes -key ca.key -subj "/CN=8.8.8.8" -days 10000 -out ca.crt
      ```

- [ ] Generate docker docker registry server key file

      ```bash
      openssl genrsa -out private-registry-server.key 2048
      ```

- [ ] Create a config file for generating a Certificate Signing Request (CSR) for  docker registry server

      ```bash
      cat > private-registry-server.conf <<EOF
      [ req ]
      default_bits = 2048
      prompt = no
      default_md = sha256
      req_extensions = req_ext
      distinguished_name = dn
          
      [ dn ]
      C = cn #replace with actual info
      ST = Tianjin #replace with actual info
      L = Tianjin #replace with actual info
      O = Air Gaped Company #replace with actual info
      OU = Air Gaped team #replace with actual info
      CN = master01.airgaped.org  #replace with actual registry server name
          
      [ req_ext ]
      subjectAltName = @alt_names
          
      [ alt_names ]
      DNS.1 = master01.airgaped.org #replace with alternative dns resolve name
      IP.1 = 192.168.122.163 #replace with alternative ip for the registry server
          
      [ v3_ext ]
      authorityKeyIdentifier=keyid,issuer:always
      basicConstraints=CA:FALSE
      keyUsage=keyEncipherment,dataEncipherment
      extendedKeyUsage=serverAuth,clientAuth
      subjectAltName=@alt_names

      EOF
      ```

- [ ] Generate the certificate signing request based on the config file for docker registry server

      ```bash
      openssl req -new -key  private-registry-server.key \
      -out private-registry-server.csr \
      -config private-registry-server.conf
      ```

- [ ] Sign the registry server certificate with CA cert and docker registry server CSR file

      ```bash
      openssl x509 -req -in private-registry-server.csr -CA ca.crt -CAkey ca.key \
      -CAcreateserial -out private-registry-server.crt -days 10000 \
      -extensions v3_ext -extfile private-registry-server.conf
      ```

- [ ] Verify the final cert for registry server

      ```bash
      openssl x509  -noout -text -in private-registry-server.crt
      ```

      ​
