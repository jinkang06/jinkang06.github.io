---
layout:     post   				    # 使用的布局（不需要改）
title:      Raspberry Pi 4B emulating Windows XP with qemu 				# 标题 
subtitle:   Including installing drivers of the system #副标题
date:       2024-04-06 				# 时间
author:     Jinkang06						# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Qemu
    - Raspberry Pi
---

## System
I used **MicroXP 0.82** to install the system. As the CPU is rather weak for a full installation of Windows XP.
## Complie Qemu 8.22
The default version of Qemu installed from apt can not assign more than one vCPU, so I decide to compile a latest version of Qemu. 
I followed <a href="https://www.chrisrcook.com/2023/09/27/building-qemu-8-0-on-raspberry-pi-os/">this blog</a>  to build the latest qemu.
Firstly, I installed the dependencies:
```
sudo apt-get install git libglib2.0-dev libfdt-dev libpixman-1-dev zlib1g-dev ninja-build libaio-dev libbluetooth-dev libcapstone-dev libbrlapi-dev libbz2-dev libcap-ng-dev libcurl4-gnutls-dev libgtk-3-dev libibverbs-dev libjpeg9-dev libncurses5-dev libnuma-dev librbd-dev librdmacm-dev libsasl2-dev libsdl2-dev libseccomp-dev libsnappy-dev libssh-dev libvde-dev libvdeplug-dev libvte-2.91-dev libxen-dev liblzo2-dev valgrind xfslibs-dev libnfs-dev bison flex libiscsi-dev
```
Then, I download the source code from qemu.org, and run `./configure  --enable-spice --target-list=x86_64-softmmu --enable-kvm && make` 

## Problems I met
### 1. `bios-256k.bin not found`, there are two reasons:
The problem occured is due to the command `sudo make install` will copy the compiled files into `/usr/local/share/qemu/`, we can use `cp -p /usr/local/share/qemu/ -r /usr/share/qemu`.
### 2. It is hard to install the disk SCSI driver in Windows XP
I download the `virtio-win-0.1.173.iso` from `https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/` . This ISO file for SCSI disk driver is usable.
### 3. If you use `virt-manager` (Libvirtd), Apparmor will stop `virt-manager` to access the files in `/usr/local/share/qemu/`
So, you should edit `/etc/apparmor.d/usr.lib.libvirt.virt-aa-helper` and `usr.sbin.libvirtd`. I added several lines into the two files:
```
/usr/local/share/qemu/* r,
/usr/local/bin/* r,
/usr/share/qemu/* r,
```
before `}`.
### 4. The myth in the number of vCPU cores:
The Windows XP guest VM will degrade its speed when I assign more than one core to it.  
If I assign two more vCPUs, I can only observe 2 cores in the VM system.  
If I assign four vCPUs (raspberry pi 4b's maximum capacity), and force the topology of CPU with 4 cores, 1 thread, and 1 socket, the VM is hard to start normally because it is too slow!
**Can any one help me about this?**
## The currently used Qemu XML
```
<domain type='qemu' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
  <name>winxp-x86_64</name>
  <uuid>fecab037-56c0-4f65-9825-144de365bc86</uuid>
  <metadata>
    <libosinfo:libosinfo xmlns:libosinfo="http://libosinfo.org/xmlns/libvirt/domain/1.0">
      <libosinfo:os id="http://microsoft.com/win/xp"/>
    </libosinfo:libosinfo>
  </metadata>
  <memory unit='KiB'>512000</memory>
  <currentMemory unit='KiB'>512000</currentMemory>
  <memoryBacking>
    <source type='memfd'/>
    <access mode='shared'/>
  </memoryBacking>
  <vcpu placement='static'>1</vcpu>
  <os>
    <type arch='x86_64' machine='pc-i440fx-7.2'>hvm</type>
    <boot dev='hd'/>
    <bootmenu enable='yes'/>
  </os>
  <features>
    <acpi/>
    <apic/>
    <hyperv mode='custom'>
      <relaxed state='on'/>
      <vapic state='on'/>
      <spinlocks state='on' retries='8191'/>
    </hyperv>
    <vmport state='off'/>
  </features>
  <cpu mode='custom' match='exact' check='none'>
    <model fallback='forbid'>qemu64</model>
    <topology sockets='1' dies='1' cores='1' threads='1'/>
  </cpu>
  <clock offset='localtime'>
    <timer name='hpet' present='yes'/>
    <timer name='hypervclock' present='yes'/>
  </clock>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>destroy</on_crash>
  <pm>
    <suspend-to-mem enabled='no'/>
    <suspend-to-disk enabled='no'/>
  </pm>
  <devices>
    <emulator>/usr/local/bin/qemu-system-x86_64</emulator>
    <disk type='file' device='disk'>
      <driver name='qemu' type='raw' io='threads'/>
      <source file='/VMs/winxp.img'/>
      <backingStore/>
      <target dev='vda' bus='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
    </disk>
    <disk type='file' device='disk'>
      <driver name='qemu' type='raw' io='threads'/>
      <source file='/VMs/vol.img'/>
      <target dev='vdb' bus='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>
    </disk>
    <controller type='usb' index='0' model='ich9-ehci1'>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x7'/>
    </controller>
    <controller type='usb' index='0' model='ich9-uhci1'>
      <master startport='0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0' multifunction='on'/>
    </controller>
    <controller type='usb' index='0' model='ich9-uhci2'>
      <master startport='2'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x1'/>
    </controller>
    <controller type='usb' index='0' model='ich9-uhci3'>
      <master startport='4'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x2'/>
    </controller>
    <controller type='pci' index='0' model='pci-root'/>
    <interface type='network'>
      <mac address='52:54:00:e4:d4:ee'/>
      <source network='default'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
    <input type='mouse' bus='ps2'/>
    <input type='keyboard' bus='ps2'/>
    <input type='tablet' bus='usb'>
      <address type='usb' bus='0' port='1'/>
    </input>
    <graphics type='vnc' port='5901' autoport='no' listen='0.0.0.0'>
      <listen type='address' address='0.0.0.0'/>
    </graphics>
    <audio id='1' type='none'/>
    <video>
      <model type='qxl' ram='65536' vram='65536' vgamem='65536' heads='1' primary='yes'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0'/>
    </video>
    <memballoon model='virtio'>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x07' function='0x0'/>
    </memballoon>
  </devices>
  <qemu:commandline>
    <qemu:arg value='-L'/>
    <qemu:arg value='/usr/share/qemu'/>
  </qemu:commandline>
</domain>


```