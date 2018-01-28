---
layout: post
title: "Deploy SaltStack in air-gapped environments"
date: 2018-01-15 23:10:01 +0800
tags: ["SaltStack"]
---

## 1. Pre-defined environment
We have 4 node and one master which is `jin`, `mu`,`shui`,`huo` and `master` with ip **192.168.122.1**.
We assume that the `master` have both the Internet access and private network access.

## 2. Download and install requested rpms
**On the master node**
```bash
mkdir salt
sudo yum install https://repo.saltstack.com/yum/redhat/salt-repo-latest-2.el7.noarch.rpm
yum clean expire-cache
yum install --downloadonly --downloaddir=./salt salt-master salt-minion salt-ssh salt-syndic salt-api

rpm -ivh ~salt/*.rpm
systemctl enable salt-minion && systemctl start salt-master

firewall-cmd --permanent --add-port=4505-4506/tcp

for host in jin mu shui huo ; do
  echo "For $host"
  ssh $host mkdir ~/salt
  scp ~/salt/* $host:~/salt
  ssh $host rpm -ivh ~/salt/*.rpm
  ssh $host "sed -i 's/#master: salt/master: 192.168.122.1/g' /etc/salt/minion"
  ssh $host "systemctl enable salt-minion && systemctl start salt-minion"
done
```

## 3. Authorize the minions joining request
If everything goes well, on the `master` node you may found the following output

```bash
salt-key --list-all
# Accepted Keys:
# Denied Keys:
# Unaccepted Keys:
# huo
# jin
# mu
# shui
# Rejected Keys:
```
Accept the  minions joining request

```bash
salt-key --accept-all
# The following keys are going to be accepted:
# Unaccepted Keys:
# huo
# jin
# mu
# shui
# Proceed? [n/Y] y
# Key for minion huo accepted.
# Key for minion jin accepted.
# Key for minion mu accepted.
# Key for minion shui accepted.
```

## 4. Fire your first cluster command

```bash
salt '*' test.ping
# mu:
#     True
# jin:
#     True
# huo:
#     True
# shui:
#     True
```

## 5. Run commands in cluster

```bash
salt '*' cmd.run 'hostname'
# shui:
#     shui
# huo:
#     huo
# mu:
#     mu
# jin:
#     jin
```
