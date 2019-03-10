---
layout: post
title:  "Automated Virtualbox VM Setup"
date:   2019-03-10
categories: start virtualbox provision
videoName: kickstart.mp4
---
## Automated Virtualbox VM Setup with cloud-init
This page is mainly following a [blog page] by Marko Vuksanovic, with slight modification.

One of interesting alternative to automate Virtualbox VM setup is by making use of cloud-init.
This approach requires manual OS installation step at the beginning, and preparation of cloud-init seed ISO image.
In this guide, I will use CentOS minimal as the base image. Provision step will be done by ansible, pulled from local HTTP server.

Steps:
- Install CentOS on virtualbox
- Install Cloud Init and Ansible
- Export VM as Virtualbox Appliance
- Prepare Cloud Init seed ISO image
- Prepare ansible scripts
- Test

### Install CentOS on virtualbox

Download CentOS ISO image from http://centos.org. In this case I use minimal ISO image.
```
wget http://isoredirect.centos.org/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1810.iso
```
Let's verify the checksum
```
$ sha256sum CentOS-7-x86_64-Minimal-1810.iso 
38d5d51d9d100fd73df031ffd6bd8b1297ce24660dc8c13a3b8b4534a4bd291c  CentOS-7-x86_64-Minimal-1810.iso
```
Compare to checksum provided in the centos website: [centos checksum].

Create Virtualbox VM, attach downloaded medium, then continue to install. Because we will need to install cloud-init, ensure the networking mode provide you an internet access.

### Install Cloud Init
Login to the Guest VM using root.
```
# yum install cloud-init -y
# yum install epel-release -y
# yum install ansible -y
```

### Export VM as Virtualbox Appliance
Turn off the VM, export via menu "File > Export Appliance".
At this point, we have an Virtualbox OVA file containing Cloud Init and Ansible package. Let's name it centos-cloud-init.ova

### Prepare Cloud Init seed ISO image
Create two files: user-data and meta-data.

user-data:
```
#cloud-config
manage_etc_hosts: true

runcmd:
  - 'curl -Os http://<host ip address>:8000/playbook.tar.gz >> /tmp/cmd.txt 2>&1' 
  - 'tar -xzvf playbook.tar.gz >> /tmp/cmd.txt 2>&1'
  - 'ansible-playbook -c local -vvvv -i jboss-standalone/hosts jboss-standalone/site.yml > /tmp/ansible-output.txt 2>&1'
```
Keep in mind that the commands listed above will be executed as root user, from directory `/`.

meta-data file:
```
hostname: my_new_vm
local-hostname: my_new_vm
```

Next, use `genisoimage` to create ISO file:
```
$ genisoimage -output seed.iso -volid cidata -R -J user-data meta-data
I: -input-charset not specified, using utf-8 (detected in locale settings)
Total translation table size: 0
Total rockridge attributes bytes: 331
Total directory bytes: 0
Path table size(bytes): 10
Max brk space used 0
183 extents written (0 MB)

$ ls -ltr seed.iso 
-rw-r--r-- 1 haikal haikal 374784 Mac  10 03:32 seed.iso

```
### Prepare ansible scripts
As an example, download the jboss playbook from [jboss-standalone] Ansible example.
Update hosts and site YAML file:
hosts file:
```
[jboss]
localhost
```

site.yml:
```
---
# This playbook deploys a simple standalone JBoss server.

- hosts: jboss

  roles:
    - jboss-standalone
```

Then compress the directory as playbook.tar.gz. In the same directory, fire up python simple HTTP server.
```
$ python -m SimpleHTTPServer
Serving HTTP on 0.0.0.0 port 8000 ...
```
Try the HTTP link provided in user-data file before continue.

We should be able to see the log printed when the cloud init pulls the tar.gz file. Should similar to:
```
192.168.123.123 - - [10/Mar/2019 03:23:28] "GET /playbook.tar.gz HTTP/1.1" 200 -
```
Else it means the ip address is not reachable from within the new VM.


### Test
Import centos-cloud-init.ova, attach the cloud init seed ISO file, ensure the network setting allow this new VM to reach your python web server. If the Ansible playbook requires internet connection, the network setting must allow this as well.
Then start the VM. Depending on the VM specification and the internet connection speed, the login prompt can appear before the cloud init finishes his task.

The result of ansible execution should be seen from `/tmp/ansible-output.txt` log file, and for cloud init, the log file is `/var/log/cloud-init.log`.

## Alternative: using kickstart

Other alternative is posted in [devopstribe blog post], which use linux kickstart.
Provided that a kickstart and an ISO linux installer file already in place, this approach can be done quickly.

A kickstart file, `/root/anaconda-ks.cfg`, is generated after a CentOS installation finished. For this exercise, I use [this kickstart] file as a sample. 

First step, go to the directory where the kickstart file is located, then fire up HTTP server, to serve the kickstart file.
```
$ python -m SimpleHTTPServer
Serving HTTP on 0.0.0.0 port 8000 ...
```

On separate terminal, test the url: `curl -O http://<Virtualbox Host-only ip address>:8000/ks.cfg`

Next create a VM in Virtualbox, create Disk, attach the minimal CentOS ISO file, and set the networking to Host only.
Upon booting, press Tab to get to the boot prompt. Then add boot parameters for kickstart address: `ksdevice=enp0s3 ip=192.168.99.101 netmask=255.255.255.0 ks=http://192.168.99.1:8000/ks.cfg`.

{% include videoPlayer.html id=page.videoName %}

## VirtualBox unattended installation
If you don't like the above two alternatives, another option to check is to use VirtualBox unattended installation, as posted on this [page] (https://kifarunix.com/how-to-automate-virtual-machine-installation-on-virtualbox/ ). This is built-in command from VirtualBox.

When I tried this approach, my VirtualBox installation complain that some preseed configuration files are missing. Googled around, and I found that these file can be downloaded from [here](https://www.virtualbox.org/svn/vbox/trunk/src/VBox/Main/UnattendedTemplates/).

[blog page]: https://medium.com/@mvuksano/automated-provisioning-of-virtual-machines-for-development-8a543e435f44
[centos checksum]: https://wiki.centos.org/Manuals/ReleaseNotes/CentOS7.1810?action=show&redirect=Manuals%2FReleaseNotes%2FCentOS7#head-216cf28780660383fed5b3266f31ef11ea95d18f
[jboss-standalone]: https://github.com/ansible/ansible-examples/tree/master/jboss-standalone.
[this kickstart]: /files/ks.cfg
[devopstribe blog post]: https://devopstribe.it/2014/07/23/install-linux-centos-7-with-kickstart-on-virtualbox/

