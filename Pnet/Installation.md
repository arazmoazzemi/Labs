

- ***Install kvm***

```
apt-get update && apt-get -y upgrade
apt-get -y install qemu-kvm libvirt-dev libvirt-daemon-system virtinst virt-manager virt-viewer libvirt-clients bridge-utils oz libguestfs-tools uuid-runtime curl linux-source xauth ssh-askpass libosinfo-bin  
systemctl enable libvirtd && systemctl start libvirtd
systemctl status libvirtd
```

- ***Configure KVM***
```
sudo usermod -aG libvirt $USER
sudo usermod -aG kvm $USER

sudo usermod -aG libvirt oneadmin
sudo usermod -aG kvm oneadmin
```

- ***show group members***

```
getent group libvirt
getent group kvm
```

- ***Enable nested virtualization***
```
# Before enabling nested VT feature, power off all running VMs.

sudo modprobe -r kvm_intel
sudo modprobe kvm_intel nested=1

# Enable Nested Virtualization Permanently

sudo nano /etc/modprobe.d/kvm.conf
options kvm_intel nested=1

cat /sys/module/kvm_intel/parameters/nested
modinfo kvm_intel | grep -i nested


# Enable Nested Feature In KVM Guests Using Virt-manager

Copy host CPU configuration

Open Virt-manager GUI application and double click the KVM guest in which you want to enable nested VT feature.
 Click on the "Show virtual hardware details" button and go to the "CPUs" section in left menu.

Select the "Copy host CPU configuration" check box in the CPU configuration window and click Apply.

egrep --color -i "svm|vmx" /proc/cpuinfo
```

virsh list --all

virsh edit ubuntu-test

virssh dumpxl ubuntu-test

Find "cpu mode" parameter and set its value as "host-model".

<cpu mode='host-model' check='partial'/>


- ***make a bidge netwok***
```
brctl show

virsh net-list --all

# destroy and undefine every bridge networks
virsh net-destroy default
virsh net-undefine default
ip link delete virbr0




#add below

nano /etc/libvirt/qemu/networks/vmbr0.xml

<network>
  <name>vmbr0</name>
  <forward mode="bridge"/>
  <bridge name="vmbr0" />
</network>


cd /etc/libvirt/qemu/networks/
sudo mv /etc/libvirt/qemu/networks/default.xml default.xml.orig
sudo rm /etc/libvirt/qemu/networks/default.xml
virsh net-define --file /etc/libvirt/qemu/networks/vmbr0.xml

virsh net-start vmbr0
virsh net-autostart --network vmbr0

service libvirtd restart


virsh net-info default

virsh net-dumpxml default

virsh net-destroy default

virsh net-start default

```
- ***Create a bridge with yaml config***

```
# default network is enp1s0

nano /etc/netplan/00-installer-config.yaml

network:
  version: 2
  renderer: networkd

  ethernets:
    eno1:
      dhcp4: false
      dhcp6: false

  bridges:
    vmbr0:
      interfaces: [eno1]
      addresses: [192.168.31.30/24]
      # gateway4 is deprecated, use routes instead
      routes:
      - to: default
        via: 192.168.31.7
        metric: 100
        on-link: true
      mtu: 1500
      nameservers:
        addresses: [8.8.8.8]
      parameters:
        stp: true
        forward-delay: 4
      dhcp4: no
      dhcp6: no



sudo netplan app

```


- ***Preparing Machine Images for qemu/kvm***
```
# PnetLab qcow2 image


scp tar -xfv PNET_4.2.10.ova <user@password:/home/it>

tar -xfv PNET_4.2.10.ova

qemu-img convert -O qcow2 PNET_4.2.10-disk1.vmdk PNET_4.2.10-disk1.qcow2

```

- ***Change PNET Lab ip address***

```
rm -f /opt/ovf/.configured
reboot

```

- ***OSINFO***

```
sudo apt install osinfo-db-tools
wget -O "/tmp/osinfo-db.tar.xz" "https://releases.pagure.org/libosinfo/osinfo-db-20230518.tar.xz"
sudo osinfo-db-import --local "/tmp/osinfo-db.tar.xz"


osinfo-query os

virt-install --name PnetLab --memory 16384 --vcpus 18 --disk /home/it/nvme01/images/PNET_4.2.10-disk1.qcow2,bus=sata --import --os-variant ubuntu18.04 --network default

```

- ***Example for install on kvm-host***

```
virt-install \
   -n vm_name \
   --connect=qemu:///system \
   --description "Any description" \
   --os-type=Linux \
   --ram=2048 \
   --vcpus=1 \
   --disk path=image_file,bus=virtio,size=12 \
   --graphics vnc \
   --network bridge=virbr0,model=virtio \
   --boot hd

```
- ***Check for ip forward***
```
sysctl -p

```

- ***Stop iptables***

```
iptables -P INPUT ACCEPT
iptables -P OUTPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -F
```



- ***Upgrade PNET Lab***

#upload files to /tmp/
5.0.1
5.2.7
5.3.11
----------------------------------------------5.0.1-------------------------------

unzip 5.0.1 -d ./upgrade > /dev/null 2>&1

chmod 755 -R upgrade  

find upgrade -type f -print0 | xargs -0 dos2unix 2>&1 > /dev/null 2>&1

./upgrade/upgrade

reboot

------------------------------------------------5.2.7------------------------------

unzip 5.2.7 -d ./upgrade > /dev/null 2>&1

chmod 755 -R upgrade  

find upgrade -type f -print0 | xargs -0 dos2unix 2>&1 > /dev/null 2>&1

./upgrade/upgrade

reboot

-----------------------------------------------5.3.11---------------------------------

unzip 5.3.11 -d ./upgrade > /dev/null 2>&1

chmod 755 -R upgrade  

find upgrade -type f -print0 | xargs -0 dos2unix 2>&1 > /dev/null 2>&1

./upgrade/upgrade

reboot


- ***Resize Disk Size***
#host

du -h /home/it/nvme01/images/PNET_4.2.10-disk1.qcow2

#shutdown guest
qemu-img info /home/it/nvme01/images/PNET_4.2.10-disk1.qcow2

qemu-img resize /home/it/nvme01/images/PNET_4.2.10-disk1.qcow2 +200G



#VM

- ***Fix disk errors***

parted -l
#type fix

lsblk -f
df -h

cfdisk -L /dev/sda
# add free space to partition disk /dev/sda3

vgdisplay 

lvdisplay 

pvresize /dev/vda3

lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv

resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv



https://packetpushers.net/ubuntu-extend-your-default-lvm-space/




-----------------------------PnetLab-Images----------------------------------
-----------------------------------------------------------------------------
https://legacy.unetlab.cloud/1:/

http://dl.nextadmin.net/dl/EVE-NG-image/qemu/
-----------------------------------------------------------------------------
------------------------------------------------------------------------------

------------imeges-----------------------
# upload images with folder name


/opt/unetlab/addons/qemu


------------iol----------------------

# upload 
i86bi_LinuxL3-AdvEnterpriseK9-M2_157_3_May_2018.bin
i86bi_LinuxL2-AdvEnterpriseK9-M_152_May_2018.bin

/opt/unetlab/addons/iol/bin

python2 /opt/unetlab/addons/iol/bin/CiscoIOUKeygen.py

/opt/unetlab/wrappers/unl_wrapper -a fixpermissions

---------------update_pnetlab----5.0.1------------

scp 5.0.1.zip root@192.168.31.33:/root

unzip 5.0.1.zip -d ./upgrade > /dev/null 2>&1

chmod 755 -R upgrade  

find upgrade -type f -print0 | xargs -0 dos2unix 2>&1 > /dev/null 2>&1

./upgrade/upgrade

reboot

-------5.2.7-----------

scp 5.2.7.zip root@192.168.31.33:/root

unzip 5.2.7.zip -d ./upgrade > /dev/null 2>&1

chmod 755 -R upgrade  

find upgrade -type f -print0 | xargs -0 dos2unix 2>&1 > /dev/null 2>&1

./upgrade/upgrade

reboot


----5.3.11-----------

scp 5.3.11.zip root@192.168.31.33:/root

unzip 5.3.11.zip -d ./upgrade > /dev/null 2>&1

chmod 755 -R upgrade  

find upgrade -type f -print0 | xargs -0 dos2unix 2>&1 > /dev/null 2>&1

./upgrade/upgrade

reboot

------------------------------------------------------








