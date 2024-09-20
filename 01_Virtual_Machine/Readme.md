
# Physical to Virtual Machine

The physical server is Server SAP Development (QA) running on RHEL5.5. The Host for virtualization is Server Proxmox 2.3.
We will migrate all datas while the guest is online using rsync ssh.

## Server Proxmox 2.3

### Creating VM

Create VM with space 1000G. We will use file iso `RHEL-Server-5.5-x86_64-dvd.iso` to install the VM.

File konfigurasi VM `/etc/pve/qemu-server/501.conf`

```
balloon: 4096
boot: cd
bootdisk: sata0
cores: 4
ide2: NFS:iso/RHEL-Server-5.5-x86_64-dvd.iso,media=cdrom,size=3616704K
memory: 8192
name: sap4.domain.id
net0: rtl8139=52:F8:DA:1D:A5:D4,bridge=vmbr0
ostype: l26
sata0: local:501/vm-501-disk-1.raw,size=1000G
sockets: 1  
```

Do instal RHEL5.5 mode text. Choose only packages core and base. Then create partition like this

- /boot	   300 MB
- swap	 16384 MB
- /

>All left space for root parition

After installing shutdown the VM.

### First Booting on VM 

We boot VM with iso Debian 7 LiveCD. After booting create directory `/media/sap4`

```sh
mkdir /media/sap4
```

then mount the sap4 partition

```sh
mount /dev/sda3 /media/sap4
```

We need to run the ssh service. Write down the ip address of the debian. In this case the ip is 192.168.20.1.

## Server SAP

### Rsync Online

On Server SAP `sap4` create ssh without login to Debian LiveCD.

```sh
ssh-keygen -f ~/.ssh/id_rsa -q -P ""
cd .ssh
```

Copy the contents of `id_rsa.pub` to Debian LiveCD on file `/root/.ssh/authorized_keys`

```
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null root@192.168.20.1
```

Do rsync for these directories

```sh
[root@sap4 ~]# ls /
*asd          *CRE_ROOT   etc    lib64        *media   opt      sbin             *srv       *tmp
*backupQNAP   *data      *hdd    logdirA      *misc   *proc     selinux          *sys        usr -> */usr/sap/trans
 bin           db2        home   logdirB      *mnt     root    *sources           testt~     var
*boot         *dev        lib   *lost+found   *net     sapmnt   sources.rb2.xml   tftpboot
```

>'*' means exclude this folder from rsync

Create script `rsync-livecd.sh`

```sh
#!/bin/bash

# IP DHCP Debian LiveCD
DSTIP=192.168.20.1
DSTDIR=/media/sap4/

for SDIR in /bin /etc /usr /home /lib /lib64 /logdirA /logdirB /opt /root /sapmnt /sbin /selinux /tftpboot /var /db2 
do
  rsync -arvpz --delete --numeric-ids --exclude=/usr/sap/trans --exclude=/etc/fstab \
    --exclude=/lib/modules/2.6.18-194.el5 --exclude=/etc/inittab --exclude=/etc/mdadm.conf \
    --exclude=/etc/modprobe.conf --exclude=/etc/sysconfig/network \
    --exclude=/etc/sysconfig/network-script/ifcfg-eth0
    -e "ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null" $SDIR $DSTIP:$DSTDIR
  echo ""
  echo "***************"
  echo ""
  sleep 1
done  
```

Execute this command at Server sap4.

```sh
nohup /root/rsync-livecd.sh &
```

## Debian LiveCD

After rsync online has finished we continue the configuration for VM sap4 on Debian LiveCD.

### Configuration for VM

Do chroot

```sh
chroot /media/sap4 /bin/bash
```

Edit file `/etc/inittab` to shutdown all consoles except tty1

```sh
sed -i -e 's/^[2-9].*getty.*tty/#&/g' /etc/inittab 
```

The VM will be running on runlevel 3 not mode multiuser (GUI)

```sh
sed -i -e 's/^\(id\):5:\(initdefault\)/\1:3:\2/g' /etc/inittab
```

Edit `/etc/sysconfig/network`

```
NETWORKING=yes
NETWORKING_IPV6=no
HOSTNAME=sap4
GATEWAY=192.168.20.254
```

Edit `/etc/sysconfig/network-script/ifcfg-eth0`

```
DEVICE=eth0
BOOTPROTO=static
IPADDR=192.168.20.100
NETMASK=255.255.255.0
ONBOOT=yes
HWADDR=00:25:90:00:DE:80
```

>Mac address must be same as the mac address of the Server sap4.

Edit `/etc/hosts`

```
# Do not remove the following line, or various programs
# that require network functionality will fail.
127.0.0.1	localhost.localdomain 	localhost
192.168.20.100	sap4.domain.id 	sap4
192.168.10.1    sap1.domain.id  sap1
192.168.10.2    sap2.domain.id  sap2
192.168.10.100	nas.domain.id	nas
::1		localhost6.localdomain6 localhost6
```

Edit `/etc/rc.local`

```sh
#!/bin/sh
#
# This script will be executed *after* all the other init scripts.
# You can put your own initialization stuff in here if you don't
# want to do the full Sys V style init stuff.

touch /var/lock/subsys/local
#su - rb1adm -c /sapmnt/RB1/exe/startsap
#su - rb2adm -c /sapmnt/RB2/exe/startsap
```

Disable unneeded services.

```sh
cd /
for I in acpid bluetooth cups hplip iscsi iscsid isdn kudzu lvm2-monitor hidd mdmonitor rhnsd rpcgssd smartd xend xendomains yum-updatesd pcscd
do
  OFF=`ls /etc/rc0.d/ | grep $I$`
  ON=`ls /etc/rc3.d/ | grep $I$`
  # remove service link
  rm -f /etc/rc3.d/$ON
  # create new service link
  cd /etc/rc3.d
  ln -s ../init.d/$I $OFF
  cd /
done
```

Exit from chroot and umount rhel partition

```sh
exit
umount /media/sap4
```

Shutdown the VM. Then set first first boot order to disk sata0.

## Booting the VM Server SAP

Boot the VM. After booting

Check the file `/etc/fstab`

```
LABEL=/                 /                       ext3    defaults        1 1
LABEL=/boot1            /boot                   ext3    defaults        1 2
tmpfs                   /dev/shm                tmpfs   size=35G        0 0
devpts                  /dev/pts                devpts  gid=5,mode=620  0 0
sysfs                   /sys                    sysfs   defaults        0 0
proc                    /proc                   proc    defaults        0 0
LABEL=SWAP-sda2         swap                    swap    defaults        0 0
```

### Removing unneeded packages via `yum remove`

Create file `/etc/yum.repos.d/rhel-dvd.repo`

```
[dvd] 
name=Red Hat Enterprise Linux Installation DVD 
baseurl=file:///media/RHEL5.5/Server 
enabled=0 
gpgcheck=1 
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
```

Mount DVD RHEL5.5.

    mkdir /media/RHEL5.5
    mount /dev/dvd /media/RHEL5.5

Tuning the VM by removing unneeded packages.

```sh
yum remove --enablerepo=dvd smartmontools 
yum remove --enablerepo=dvd acpid 
yum remove --enablerepo=dvd bluez-utils 
yum remove --enablerepo=dvd system-config-network-tui 
yum remove --enablerepo=dvd kudzu 
yum remove --enablerepo=dvd mdadm 
yum remove --enablerepo=dvd esc 
yum remove --enablerepo=dvd ccid 
yum remove --enablerepo=dvd pcsc-lite 
yum remove --enablerepo=dvd mcelog 
yum remove --enablerepo=dvd mkinitrd 
yum remove --enablerepo=dvd lvm2 
yum remove --enablerepo=dvd bluez-gnome 
yum remove --enablerepo=dvd isdn4k-utils 
yum remove --enablerepo=dvd yum-updatesd 
yum remove --enablerepo=dvd iptables-ipv6 
yum remove --enablerepo=dvd wpa_supplicant 
yum remove --enablerepo=dvd hplip 
yum remove --enablerepo=dvd setroubleshoot-plugins
yum groupremove --enablerepo=dvd sound-and-video 
yum groupremove --enablerepo=dvd x-software-development
yum groupremove --enablerepo=dvd office 
yum remove --enablerepo=dvd hpijs 
yum remove --enablerepo=dvd bluez-hcidump 
yum remove --enablerepo=dvd cadaver 
yum remove --enablerepo=dvd slrn 
yum remove --enablerepo=dvd emacs
yum remove --enablerepo=dvd emacs-leim
yum remove --enablerepo=dvd psgml
yum remove --enablerepo=dvd joystick
yum remove --enablerepo=dvd ImageMagick
yum remove --enablerepo=dvd dcraw
yum groupremove --enablerepo=dvd printing
yum groupremove --enablerepo=dvd gnome-desktop
yum groupremove --enablerepo=dvd graphical-internet
yum groupremove --enablerepo=dvd server-cfg
yum remove --enablerepo=dvd kernel-devel
```

### Database SAP

#### Turn on Database

Login as rb1adm

>At first it needs more than 30 minutes to completed.

```sh
[root@sap4 ~]# su - rb1adm

sap4:rb1adm 51> startsap db sap4

Checking db6 db Database
------------------------------
 Database is not available via R3trans
 Running /usr/sap/RB1/SYS/exe/run/startdb 
03/20/2015 08:05:33     0   0   SQL1063N  DB2START processing was successful.
SQL1063N  DB2START processing was successful.

Database activated
 /usr/sap/RB1/SYS/exe/run/startdb completed successfully 

Starting SAP-Collector Daemon 
------------------------------
******************************************************************************
* This is Saposcol Version COLL 20.95 701 - v2.00, AMD/Intel x86_64 with Linux, 2007/02/16
* Usage:  saposcol -l: Start OS Collector 
*         saposcol -k: Stop  OS Collector 
*         saposcol -d: OS Collector Dialog Mode
*         saposcol -s: OS Collector Status
* Starting collector (create new process)
******************************************************************************
 saposcol on host sap4 started

sap4:rb1adm 52> 
sap4:rb1adm 52> startsap r3 sap4

Checking db6 db Database
------------------------------
 Database is running

Starting Startup Agent sapstartsrv
-----------------------------
 Instance Service on host sap4 started

Starting SAP Instance DVEBMGS00
------------------------------
 Startup-Log is written to /home/rb1adm/startsap_DVEBMGS00.log
 Instance on host sap4 started

Starting SAP-Collector Daemon 
------------------------------
***********************************************************************
* This is Saposcol Version COLL 20.95 701 - v2.00, AMD/Intel x86_64 with Linux, 2007/02/16
* Usage:  saposcol -l: Start OS Collector 
*         saposcol -k: Stop  OS Collector 
*         saposcol -d: OS Collector Dialog Mode
*         saposcol -s: OS Collector Status
* The OS Collector (PID 5763) is already running ..... 
************************************************************************
 saposcol already running

sap4:rb1adm 53> exit
logout
```

#### Database Checking

Login as rb1adm

```sh
sap4:rb1adm 51> startsap check sap4

Checking db6 db Database
------------------------------
 Database is running

Checking SAP RB1 Instance DVEBMGS00
------------------------------
 Instance DVEBMGS00 is running 
```

#### Saposcol status

Login as rb1adm

```sh
sap4:rb1adm 53> saposcol -s
**************************************************************
Collector Versions : 
  running : COLL 20.95 701 - v2.00, AMD/Intel x86_64 with Linux 
  dialog  : COLL 20.95 701 - v2.00, AMD/Intel x86_64 with Linux, 2007/02/16 
Shared Memory       : attached
Number of records   : 4908 
Active Flag         : active (01) 
Operating System    : Linux sap4 2.6.18-194.el5 #1 SMP Tue Mar 16 21: 
Collector PID       : 5763 (00001683)
Collector           : running
Start time coll.    : Fri Mar 20 08:39:21 2015

Current Time        : Fri Mar 20 09:12:04 2015

Last write access   : Fri Mar 20 09:12:00 2015

Last Read  Access   : Fri Mar 20 09:10:38 2015

Collection Interval : 10 sec (next delay).
Collection Interval : 10 sec (last ).
Status              : free 
Collect Details     : required 
Refresh             : required

Header Extention Structure
Number of x-header      Records : 1 
Number of Communication Records : 60 
Number of free Com.     Records : 60 
Resulting offset to 1.data rec. : 61 

Trace level             : 2 

Collector in IDLE - mode ? : NO
  become idle after 300 sec without read access.
  Length of Idle Interval  : 60 sec 
  Length of norm.Interval  : 10 sec 
**************************************************************
```

#### Shutdown Database

Login as rb1adm

```sh
sap4:rb1adm 54> stopsap sap4

Checking db6 db Database
------------------------------
 Database is running

Stopping the SAP instance DVEBMGS00
----------------------------------
 Shutdown-Log is written to /home/rb1adm/stopsap_DVEBMGS00.log
 Instance on host sap4 stopped
 Waiting for cleanup of resources..................
 Running /usr/sap/RB1/SYS/exe/run/stopdb 
Database is running
Continue with stop procedure
DB20000I  The DEACTIVATE DATABASE command completed successfully.
03/20/2015 09:13:28     0   0   SQL1064N  DB2STOP processing was successful.
SQL1064N  DB2STOP processing was successful.
Database sucessfully stopped
 /usr/sap/RB1/SYS/exe/run/stopdb completed successfully 

Checking db6 db Database
------------------------------
 Database is not available via R3trans
```

#### Turn on Database

Login as rb1adm

>Execution is running faster than before.

```sh
sap4:rb1adm 51> startsap sap4

Checking db6 db Database
------------------------------
 Database is not available via R3trans
 Running /usr/sap/RB1/SYS/exe/run/startdb 
03/20/2015 09:26:00     0   0   SQL1063N  DB2START processing was successful.
SQL1063N  DB2START processing was successful.
Database activated
 /usr/sap/RB1/SYS/exe/run/startdb completed successfully 

Starting Startup Agent sapstartsrv
-----------------------------
 Instance Service on host sap4 started

Starting SAP Instance DVEBMGS00
------------------------------
 Startup-Log is written to /home/rb1adm/startsap_DVEBMGS00.log
 Instance on host sap4 started

Starting SAP-Collector Daemon 
------------------------------
******************************************************************************
* This is Saposcol Version COLL 20.95 701 - v2.00, AMD/Intel x86_64 with Linux, 2007/02/16
* Usage:  saposcol -l: Start OS Collector 
*         saposcol -k: Stop  OS Collector 
*         saposcol -d: OS Collector Dialog Mode
*         saposcol -s: OS Collector Status
* Starting collector (create new process)
******************************************************************************
 saposcol on host sap4 started

sap4:rb1adm 52> date
Fri Mar 20 09:27:13 WIT 2015
sap4:rb1adm 53> 
```




