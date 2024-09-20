
# Physical to Paravirtualization

The physical server is Server SAP Development (QA) running on RHEL5.5. The Host for virtualization is Server Xen 4.1.4 on Debian Wheezy (7.11).
We will migrate all datas while the guest is online using rsync ssh.

## Server Xen 4.1.4 (server25)

Server Xen has an ip address 192.168.20.253.

### Creating LVM

At host create LVM sap4 with volume group VG1.

	lvcreate -L950G -nsap4 VG1
	lvcreate -L16G -nsap4-swap VG1

Make filesystem

	mkfs.ext3 /dev/VG1/sap4
	mkswap -f /dev/VG1/sap4-swap

Create directory `/media/sap4`

```sh
mkdir /media/sap4
```

Then mount the sap4 partition

```sh
mount /dev/VG1/sap4 /media/sap4
```

## Server SAP (sap4)

### Rsync Online

On Server SAP `sap4` create ssh without login to Server Xen.

```sh
ssh-keygen -f ~/.ssh/id_rsa -q -P ""
cd .ssh
```

Copy the contents of `id_rsa.pub` to Server Xen on file `/root/.ssh/authorized_keys`

```sh
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null root@192.168.20.253
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

# IP Server Xen
DSTIP=192.168.20.253
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

## Server Xen 4.1.4 (server25)

After rsync online has finished we continue the configuration for DomU sap4 on Server Xen.

### Configuration for Guest DomU (Paravirtualization)

Mount LVM sap4 if not mounted yet

```sh
mount /dev/VG1/sap4 /media/sap4
```

Mount the DVD `RHEL-Server-5.5-x86_64-dvd.iso` at the host and copy the kernel-xen to `/media/sap4`

```sh
mkdir /mnt/dvd
mount -o loop,ro /usr/local/src/iso/RHEL-Server-5.5-x86_64-dvd.iso /mnt/dvd
cp /mnt/dvd/Server/kernel-xen-2.6.18-194.el5.x86_64.rpm /media/sap4
umount /mnt/dvd
```

Edit file `/media/sap4/etc/mtab`

```
proc /proc proc rw 0 0
```
 
Do chroot

```sh
chroot /media/sap4/ /bin/bash
```

Remove 

```sh
rm -rf /dev/mapper/
rm -rf /dev/VG00/
rm -f /dev/null
```

>If the Server sap4 has a LVM then remove all device files which link to LVM

Create some device file at dev directory

```sh
mknod -m 600 /dev/console c 5 1
mknod -m 666 /dev/null c 1 3
mknod -m 666 /dev/zero c 1 5
mknod -m 666 /dev/tty c 5 0
mknod -m 666 /dev/random c 1 8
mknod -m 444 /dev/urandom c 1 9
cd /dev
ln -sf null XOR
```

Enter command below to test if rpm tool is working well.

```sh
rpm -qa
```

Before install kernel-xen we could create some device at /dev directory to surpress the output error.

```sh
mkdir /dev/pts
mkdir /dev/shm
chmod 1777 /dev/shm
cd /
```
    
Do install kernel-xen test mode

```sh
[root@server25 /]# rpm -ivh --test kernel-xen-2.6.18-194.el5.x86_64.rpm 
Preparing...                ########################################### [100%]
    package kernel-xen-2.6.18-194.el5.x86_64 is already installed
```

Actually package kernel-xen already installed.

Create /sys/block

```sh
mkdir /sys/block
```

Create ramdisk

```sh
cd /
mkinitrd --omit-scsi-modules --omit-raid-modules --omit-lvm-modules --with=xennet --with=xenblk \
   --preload=xenblk /root/initrd-2.6.18-194.el5xen.img  2.6.18-194.el5xen
```

Show the ramdisk

```sh
[root@server25 ~]# ls -l /root/initrd-2.6.18-194.el5xen.img 
-rw------- 1 root root 2509141 Mar  1 15:07 /root/initrd-2.6.18-194.el5xen.img
```

Still in chroot environment. Remove the /boot directory

```sh
cd /
mv boot boot.orig
mkdir boot
```

Edit the file `/etc/fstab`

```
LABEL=/                 /                       ext3    defaults        1 1
LABEL=/boot1            /boot                   ext3    defaults        1 2
tmpfs                   /dev/shm                tmpfs   size=35G        0 0
devpts                  /dev/pts                devpts  gid=5,mode=620  0 0
sysfs                   /sys                    sysfs   defaults        0 0
proc                    /proc                   proc    defaults        0 0
LABEL=SWAP-sda2         swap                    swap    defaults        0 0
```

Edit `/etc/securetty`

    echo xvc0 >> /etc/securetty

Edit `/etc/inittab`. Change the line

    id:5:initdefault:

into

    id:3:initdefault:

Then uncomment all the lines below

    1:2345:respawn:/sbin/mingetty tty1
    2:2345:respawn:/sbin/mingetty tty2
    3:2345:respawn:/sbin/mingetty tty3
    4:2345:respawn:/sbin/mingetty tty4
    5:2345:respawn:/sbin/mingetty tty5
    6:2345:respawn:/sbin/mingetty tty6

Also add line

    co:2345:respawn:/sbin/agetty xvc0 9600 vt100-nav
    
Edit file `/etc/hosts`

    127.0.0.1       localhost.localdomain   localhost
    192.168.20.20   sap4.ricobana.co.id  sap4
    #::1            localhost6.localdomain6 localhost6

Edit file `/etc/hosts`

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

Edit `/etc/rc.local`. Only this line which is uncomment

    touch /var/lock/subsys/local

Edit `/etc/resolv.conf`

    domain domain.id
    nameserver 192.168.10.254

Edit `/etc/sysconfig/network`

    NETWORKING=yes
    NETWORKING_IPV6=no
    HOSTNAME=sap4
    GATEWAY=192.168.20.254

Edit `/etc/sysconfig/network-script/ifcfg-eth0`

    DEVICE=eth0
    BOOTPROTO=static
    IPADDR=192.168.20.100
    NETMASK=255.255.255.0
    ONBOOT=yes
    HWADDR=00:25:90:00:DE:80

>Mac address must be same as the mac address of the Server sap4.

Remove file configuration smartd and mdadm if any.

```sh
rm -f /etc/mdadm.conf
rm -f /etc/smartd.conf
```
Disable crontab for temporary by editing file `/var/spool/cron/root`.

Execute this script `sysinit.clear.sh`

```
#!/bin/bash
#
# sysinit.clear.sh
#

for I in microcode_ctl lvm2-monitor kudzu iscsid ip6tables mcstrans isdn \
   restorecond cpuspeed irqbalance iscsi mdmonitor rpcidmapd rpcgssd \
   messagebus bluetooth pcscd acpid haldaemon hidd hplip cups rawdevices \
   gpm sapinit libvirtd rhnsd avahi-daemon xend firstboot smartd xendomains
do
    OFF=`ls /etc/rc0.d/ | egrep -o "K..${I}$"`
    ON=`ls /etc/rc3.d/ | egrep -o "S..${I}$"`
    # remove service link
    echo "rm -f /etc/rc3.d/$ON"
    rm -f /etc/rc3.d/$ON
    # create new service link
    cd /etc/rc3.d
    echo "cd /etc/rc3.d; ln -s ../init.d/$I $OFF"
    ln -s ../init.d/$I $OFF
    cd /
    echo ""
done
```

Check which services are on.

```sh
[root@server25 ~]# chkconfig --list|grep 3:on
anacron        	0:off	1:off	2:on	3:on	4:on	5:on	6:off
atd            	0:off	1:off	2:off	3:on	4:on	5:on	6:off
auditd         	0:off	1:off	2:on	3:on	4:on	5:on	6:off
autofs         	0:off	1:off	2:off	3:on	4:on	5:on	6:off
crond          	0:off	1:off	2:on	3:on	4:on	5:on	6:off
iptables       	0:off	1:off	2:on	3:on	4:on	5:on	6:off
netfs          	0:off	1:off	2:off	3:on	4:on	5:on	6:off
network        	0:off	1:off	2:on	3:on	4:on	5:on	6:off
nfslock        	0:off	1:off	2:off	3:on	4:on	5:on	6:off
portmap        	0:off	1:off	2:off	3:on	4:on	5:on	6:off
readahead_early	0:off	1:off	2:on	3:on	4:on	5:on	6:off
sendmail       	0:off	1:off	2:on	3:on	4:on	5:on	6:off
setroubleshoot 	0:off	1:off	2:off	3:on	4:on	5:on	6:off
sshd           	0:off	1:off	2:on	3:on	4:on	5:on	6:off
syslog         	0:off	1:off	2:on	3:on	4:on	5:on	6:off
vncserver      	0:off	1:off	2:on	3:on	4:on	5:on	6:off
xfs            	0:off	1:off	2:on	3:on	4:on	5:on	6:off
xinetd         	0:off	1:off	2:off	3:on	4:on	5:on	6:off
```

Exit from chroot and create booting directory for DomU sap4

```sh
exit
mkdir /boot/sap4
cd /boot/sap4
cp /media/sap4/boot.orig/vmlinuz-2.6.18-194.el5xen .
cp /media/sap4/root/initrd-2.6.18-194.el5xen.img .
```

Create config file `/etc/xen/sap4.cfg`

```
#
# sap4
# RHEL 5.5
#
kernel  = '/boot/sap4/vmlinuz-2.6.18-194.el5xen'
ramdisk = '/boot/sap4/initrd-2.6.18-194.el5xen.img'
vcpus = '2'
memory = '4096'

root = "/dev/xvda"

disk = [
        'phy:/dev/VG1/sap4,xvda,w',
        'phy:/dev/VG1/sap4-swap,xvdb,w',
        'file:/usr/local/src/iso/RHEL.5-5.iso,xvdc:cdrom,r'
       ]

name = 'sap4'

# mac address must be equal with the physical sap4
vif  = [ 'bridge=xenbr0 ,mac=00:25:90:00:de:80' ]

# set the real time clock to local time [default=0 i.e. set to utc]
localtime=0

# set the real time clock offset in seconds [default=0 i.e. same as dom0]
rtc_timeoffset=0

on_poweroff = 'destroy'
on_reboot   = 'restart'
on_crash    = 'restart'
extra = "fastboot"
```

Umount and start DomU sap4

```sh
umount /media/sap4
xm create -c /etc/xen/sap4.cfg 
```

## Login into DomU Server sap4

You can login directly via console on Xen Host or via ssh.

### Create DVD Repo

Create file /etc/yum.repos.d/rhel-dvd.repo

```
[dvd] 
name=Red Hat Enterprise Linux Installation DVD 
baseurl=file:///media/RHEL5.5/Server 
enabled=0 
gpgcheck=1 
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
```

Make directory for mounting dvd redhat

```sh
mkdir /media/RHEL5.5
mount -o loop,ro /sources/RHEL-Server-5.5-x86_64-dvd.iso /media/RHEL5.5
```

### Uninstall Packages

Uninstal some packages using the script uninstall-packages.sh below

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
yum groupremove --enablerepo=dvd printing
yum groupremove --enablerepo=dvd gnome-desktop
yum groupremove --enablerepo=dvd graphical-internet
yum groupremove --enablerepo=dvd server-cfg
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
yum remove --enablerepo=dvd kernel-devel
yum remove --enablerepo=dvd pcmciautils
yum remove --enablerepo=dvd systemtap-runtime
yum remove --enablerepo=dvd mkbootdisk
yum remove --enablerepo=dvd bluez-libs
yum remove --enablerepo=dvd xen-libs
yum remove --enablerepo=dvd iscsi-initiator-utils
yum remove --enablerepo=dvd irda-utils
```

### SAP

Reactivate service sapinit

```sh
chkconfig --level 345 sapinit on
chkconfig --level 0126 sapinit off
```

### VNC

Service vncserver is run by root. Setup vncpassword with command `vncpasswd`

Edit file `/etc/sysconfig/vncservers`

```
VNCSERVERS="1:root"
VNCSERVERARGS[1]="-geometry 1024x768"
```

Start service vncserver

```sh
/etc/init.d/vncserver start
chkconfig --level 345 vncserver on
chkconfig --level 0126 vncserver off    
```

Reboot Server sap4.


