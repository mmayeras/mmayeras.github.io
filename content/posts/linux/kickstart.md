---
title: "Sample kickstart"
date: 2022-12-13
categories:
- linux
tags:
- linux
- kickstart
---

A simple kickstart for easy deploy
<!--more-->
```
lang en_US
keyboard --xlayouts='us'
timezone America/New_York --utc
rootpw $2b$10$UP68Yb7JjnLcZyELbFL2FeH5T/jIcycmorWsiqstkJOU9zbt621Jm --iscrypted
reboot
text
cdrom
bootloader --append="rhgb quiet crashkernel=1G-4G:192M,4G-64G:256M,64G-:512M"
zerombr
clearpart --all --initlabel
autopart
skipx
firstboot --disable
selinux --enforcing
firewall --enabled --ssh
user --name=skynet --groups=wheel  --plaintext --password=password
sshkey --username=skynet "ssh-rsa "
%post
echo "skynet ALL=(ALL)   NOPASSWD:ALL" >> /etc/sudoers
%end
%packages
@^minimal-environment
kexec-tools
%end
```