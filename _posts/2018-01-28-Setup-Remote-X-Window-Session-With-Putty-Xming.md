---
layout: post
title: "Setup remote x-window session with putty & xmin"
date: 2018-01-28 00:40:01 +0800
tags: ["X-window","Linux"]
---

```bash
#Setup X window environment and core Gnome component
yum -y groupinstall "X Window System"
yum install gnome-classic-session gnome-terminal nautilus-open-terminal control-center liberation-mono-fonts

#Edit /etc/ssh/sshd_config
#Set the following two options:
#X11Forwarding yes
#X11UseLocalhost no
system reload sshd
systemctl restart sshd

yum install libXScrnSaver xauth

startx /usr/bin/ssh -f -Y root@75.75.242.42 gnome-session -- :0 -clipboard

#If you encounter error like this:
# ssh_askpass: exec(/usr/sbin/ssh-askpass): No such file or directory
# Permission denied, please try again.
# ssh_askpass: exec(/usr/sbin/ssh-askpass): No such file or directory
# Permission denied, please try again.
# ssh_askpass: exec(/usr/sbin/ssh-askpass): No such file or directory
# root@75.75.242.42: Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password).
# xinit: connection to X server lost

#Please config ssh password-less by add local & remote user pub_key to remote host .ssh/authorized_keys
#Set up file permissions properly in both local and remote servers
chmod 700 /home/<user>/.ssh
chmod 644 /home/<user>/.ssh/authorized_keys

```
