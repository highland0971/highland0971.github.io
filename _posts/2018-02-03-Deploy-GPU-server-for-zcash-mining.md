---
layout: post
title: "Deploy GPU server for zcash cluster mining"
date: 2018-02-03 23:40:01 +0800
tags: ["zcash","Linux","GPU"]
---

## Setup ssh-passwd-free environment
- On host ***hb5-42***
```bash
ssh-keygen
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

cat >> /etc/hosts << EOF
172.24.223.42 hb5-42
172.24.223.40 hb5-40
172.24.223.41 hb5-41
172.24.223.43 hb5-43
172.24.223.44 hb5-44
172.24.223.45 hb5-45
EOF

for node in hb5-40 hb5-41 hb5-43 hb5-44 hb5-45 ; do
  scp ~/.ssh/* root@$node:~/.ssh/
  scp /etc/hosts root@$node:/etc/hosts
done
```

## Setup Nvidia GPU driver and CUDA driver
- On host ***hb5-42***
```bash
for node in hb5-40 hb5-41 hb5-43 hb5-44 hb5-45 hb5-42 ; do
  ssh $node "wget http://us.download.nvidia.com/tesla/390.12/nvidia-diag-driver-local-repo-ubuntu1604-390.12_1.0-1_amd64.deb " && \
  ssh $node "dpkg -i nvidia-diag-driver-local-repo-ubuntu1604-390.12_1.0-1_amd64.deb" && \
  ssh $node "apt-key add /var/nvidia-diag-driver-local-repo-390.12/7fa2af80.pub" && \
  ssh $node "apt-get update" && \
  ssh $node "apt-get install -y --no-install-recommends nvidia-modprobe cuda-drivers  --allow-unauthenticated" && \
  ssh $node "reboot" && \
done
```

```bash
#Verify with nvidia-smi
for node in hb5-40 hb5-41 hb5-43 hb5-44 hb5-45 hb5-42 ; do
  echo $node GPU detect
  ssh $node "nvidia-smi"
done
#Sat Feb  3 23:17:14 2018
#+-----------------------------------------------------------------------------+
# | NVIDIA-SMI 390.12                 Driver Version: 390.12                    |
# |-------------------------------+----------------------+----------------------+
# | GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
# | Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
# |===============================+======================+======================|
# |   0  Tesla P100-PCIE...  Off  | 00000000:00:08.0 Off |                    0 |
# | N/A   52C    P0   160W / 250W |    719MiB / 16280MiB |    100%      Default |
# +-------------------------------+----------------------+----------------------+
#
# +-----------------------------------------------------------------------------+
# | Processes:                                                       GPU Memory |
# |  GPU       PID   Type   Process name                             Usage      |
# |=============================================================================|
# |    0       955      C   /root/zm_0.5.8/zm                            709MiB |
# +-----------------------------------------------------------------------------+
```

## Setup miner binary and service
- Download ZCash miner from [[ANN] dstm's ZCash / Equihash Nvidia Miner v0.5.8 (Linux / Windows)](https://bitcointalk.org/index.php?topic=2021765.0) to host ***hb5-42***
```bash
for node in hb5-40 hb5-41 hb5-43 hb5-44 hb5-45 hb5-42 ; do
  scp zm_0.5.8.tar.gz root@$node:~/
  ssh $node "tar -zxf zm_0.5.8.tar.gz"
done
```
- Setup miner service `vi /etc/init.d/Miner` on host ***hb5-44***:
```vi
#!/bin/bash
### BEGIN INIT INFO
#
# Provides:  miner
# Required-Start:
# Required-Stop:
# Default-Start:    2 3 4 5
# Default-Stop:     0 1 6
# Short-Description:    initscript
# Description:  This file should be used to construct scripts to be placed in /etc/init.d.
#
### END INIT INFO

## Fill in name of program here.
PROG="zm"
PROG_PATH="/root/zm_0.5.8"
POOL_DEFAULT="cn1-zcash.flypool.org"
POOL_FAILOVER="asia1-zcash.flypool.org"
TADDR="t1aHMFYc5CfD6z9BRyUKgaTmNzgnRgsLJrL"
PROG_ARGS="--server $POOL_DEFAULT --port 3333 --pass x --user $TADDR.`hostname` --logfile=/root/zm_0.5.8/zm.log"
PROG_ARGS_FAILOVER="--server $POOL_FAILOVER --port 3333 --pass x --user $TADDR.`hostname` --logfile=/root/zm_0.5.8/zm.log"
PID_PATH="/var/run/"


start() {
    if [ -e "$PID_PATH/$PROG.pid" ]; then
        ## Program is running, exit with error.
        echo "Error! $PROG is currently running!" 1>&2
        exit 1
    else
        ## Change from /dev/null to something like /var/log/$PROG if you want to save output.
        $PROG_PATH/$PROG $PROG_ARGS 2>&1 >/var/log/$PROG || $PROG_PATH/$PROG $PROG_ARGS_FAILOVER 2>&1 >/var/log/$PROG &
        echo "$PROG started"
    fi
}

stop() {
    echo "begin stop"
    killall $PROG
    echo "$PROG stopped"
}

## Check to see if we are running as root first.
## Found at http://www.cyberciti.biz/tips/shell-root-user-check-script.html
if [ "$(id -u)" != "0" ]; then
    echo "This script must be run as root" 1>&2
    exit 1
fi

case "$1" in
    start)
        start
        exit 0
    ;;
    stop)
        stop
        exit 0
    ;;
    reload|restart|force-reload)
        stop
        start
        exit 0
    ;;
    **)
        echo "Usage: $0 {start|stop|reload}" 1>&2
        exit 1
    ;;
esac
```
- Enable & Start service
```bash
for node in hb5-40 hb5-41 hb5-43 hb5-44 hb5-45 ;do
scp /etc/init.d/miner $host:/etc/init.d/
ssh $host "update-rc.d miner defaults"
reboot
done
```

## XMR mining plus
```bash
wget https://cn.minergate.com/download/deb-cli -o minergate-cli-release.deb
for node in hb5-41 hb5-42 hb5-43 hb5-44 hb5-45 ;do
  scp minergate-cli-release.deb root@$node:~/
  ssh $node "dpkg -i /root/minergate-cli-release.deb"
  scp /etc/init.d/miner_xmr root@$node:/etc/init.d/
  ssh root@$node "update-rc.d miner_xmr defaults"
  ssh root@$node service miner_xmr start
done

```
