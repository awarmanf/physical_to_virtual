
# Virtualization

Virtualization is a system or a method of dividing computer resources into multiple isolated environments. It 
is possible to distinguish four types of such virtualization: emulation, paravirtualization, operating system- 
level virtualization, and multiserver (cluster) virtualization. The last approach is out of scope of this report.

Emulation makes it possible to run any non-modified operating system which supports the platform being 
emulated. Implementations in this category range from pure emulators (like Bochs) to solutions which let 
some code to be executed on the CPU natively, in order to increase performance. The main disadvantages 
of emulation are low performance and low density. Examples: VMware products, KVM, QEmu, Bochs, Parallels.

Paravirtualization is a technique to run multiple modified OSs on top of a thin layer called a hypervisor, or 
virtual machine monitor. Paravirtualization has better performance compared to emulation, but the 
disadvantage is that the _guest_ OS needs to be modified. Examples: Xen, UML. Operating system-level 
virtualization enables multiple isolated execution environments within a single operating system kernel. 
It has the best possible (i.e. close to native) performance and density, and features dynamic resource 
management. On the other hand, this technology does not allow to run different kernels
from different OSs at the same time. Examples: FreeBSD Jail, Solaris Zones/Containers, Linux-VServer,
OpenVZ and Virtuozzo.

## Physical to Virtual Migration

In this project we migrate the Server SAP Development (QA) to two kind types of virtualization: Virtual Machine 
and Paravirtualization.

### Physical to Virtual Machine

The physical server is Server SAP Development (QA) running on RHEL5.5. The Host for virtualization is 
Server Proxmox 2.3. We will migrate all datas while the guest is online using rsync ssh.

[Physical to Virtual Machine](01_Virtual_Machine/Readme.md)

### Physical to Paravirtualization

The physical server is Server SAP Development (QA) running on RHEL5.5. The Host for virtualization is 
Server Xen 4.1.4 on Debian Wheezy (7.11). We will migrate all datas while the guest is online using rsync ssh.

[Paravirtualization](02_Paravirtualization/Readme.md)

