---
layout: post
title: "Hack Yum repository for Readhat Enterprise Linux 7"
date: 2018-01-24 23:10:01 +0800
tags: ["Redhat","Linux"]
---

## Hack yum repo for rehl7

```bash
rpm -qa |grep yum
rpm -qa|grep yum|xargs rpm -e --nodeps
rpm -qa |grep yum

rpm -ivh http://mirrors.163.com/centos/7.4.1708/os/x86_64/Packages/yum-3.4.3-154.el7.centos.noarch.rpm
rpm -ivh http://mirrors.163.com/centos/7.4.1708/os/x86_64/Packages/yum-metadata-parser-1.1.4-10.el7.x86_64.rpm
rpm -ivh http://mirrors.163.com/centos/7.4.1708/os/x86_64/Packages/yum-plugin-fastestmirror-1.1.31-42.el7.noarch.rpm
rpm -ivh http://mirrors.163.com/centos/7.4.1708/os/x86_64/Packages/yum-utils-1.1.31-42.el7.noarch.rpm
rpm -ivh http://mirrors.163.com/centos/7.4.1708/os/x86_64/Packages/python-kitchen-1.1.1-5.el7.noarch.rpm
rpm -ivh http://mirrors.163.com/centos/7.4.1708/os/x86_64/Packages/python-chardet-2.2.1-1.el7_1.noarch.rpm
rpm -ivh http://mirrors.163.com/centos/7.4.1708/os/x86_64/Packages/python-kitchen-1.1.1-5.el7.noarch.rpm
rpm -ivh http://mirrors.163.com/centos/7.4.1708/os/x86_64/Packages/yum-utils-1.1.31-42.el7.noarch.rpm
rpm -ivh http://mirrors.163.com/centos/7.4.1708/os/x86_64/Packages/python-urlgrabber-3.10-8.el7.noarch.rpm
curl -L http://mirrors.163.com/.help/CentOS7-Base-163.repo -o /etc/yum.repos.d/CentOS-Base.repo
sed -i 's/$releasever/7.4.1708/g' /etc/yum.repos.d/CentOS-Base.repo

yum clean all
yum makecache
yum update

```

## Setup PXE boot session for automatic REHL7 installation
```bash
systemctl stop firewalld

yum install tftp tftp-server dhcp vsftpd syslinux xinetd system-config-kickstart
yum -y groupinstall "X Window System"
yum install gnome-classic-session gnome-terminal nautilus-open-terminal control-center liberation-mono-fonts

mkdir /var/ftp/pub/dvd
chmod 777 /var/ftp/pub/dvd
mount /dev/sr0 /var/ftp/pub/dvd
systemctl enable vsftpd
systemctl start vsftpd
systemctl status vsftpd
# ● vsftpd.service - Vsftpd ftp daemon
#    Loaded: loaded (/usr/lib/systemd/system/vsftpd.service; enabled; vendor preset: disabled)
#    Active: active (running) since Wed 2018-01-24 11:11:02 EST; 6s ago
#   Process: 22963 ExecStart=/usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf (code=exited, status=0/SUCCESS)
#  Main PID: 22964 (vsftpd)
#    CGroup: /system.slice/vsftpd.service
#            └─22964 /usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf

pwd
#/etc/dhcp
cat dhcpd.conf |grep -v '#'
# default-lease-time 600;
# max-lease-time 7200;
# log-facility local7;
# subnet 192.168.56.0 netmask 255.255.255.0 {
#   range 192.168.56.100 192.168.56.200;
#   next-server 192.168.56.2;
#   filename "pxelinux.0";
# }
systemctl enable dhcpd
systemctl start dhcpd
systemctl status dhcpd

vi /etc/xinetd.d/tftp
# service tftp
# {
#         socket_type             = dgram
#         protocol                = udp
#         wait                    = yes
#         user                    = root
#         server                  = /usr/sbin/in.tftpd
#         server_args             = -s /var/lib/tftpboot
#         disable                 = no
#         per_source              = 11
#         cps                     = 100 2
#         flags                   = IPv4
# }

cd /var/lib/tftpboot
cp /usr/share/syslinux/pxelinux.0 pxelinux.0
mkdir pxelinux.cfg
cp /var/ftp/pub/dvd/isolinux/isolinux.cfg pxelinux.cfg/default
chmod 644 pxelinux.cfg/default

cp /var/ftp/pub/dvd/isolinux/* ./

systemctl enable xinetd
systemctl start xinetd
systemctl status xinetd

systemctl enable tftp
systemctl start tftp
systemctl status tftp

system-config-kickstart #Save as ks.cfg
cp /root/ks.cfg /var/ftp/pub/

vi /var/lib/tftpboot/pxelinux.cfg/default
# append initrd=initrd.img inst.stage2=hd:LABEL=RHEL-7.4\x20Server.x86_64 quiet ks=ftp://192.168.56.2/pub/ks.cfg


yum install httpd
systemctl enable httpd
systemctl start httpd
mount /dev/sr0 /var/www/html/

```
