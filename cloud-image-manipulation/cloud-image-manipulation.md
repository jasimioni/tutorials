### Cloud Image Manipulation

*Description*: Many times we want to do changes in cloud images. There are many ways to accomplish this. This example will use qemu-nbd (Network Block Device Server) to mount the image filesystem and provide the steps to modify an Ubuntu cloud image to use LVM instead of full partitions.

#### Download a valida cloud image:

```
wget https://cloud-images.ubuntu.com/resolute/current/resolute-server-cloudimg-amd64.img
```

#### Install the required packages

```
sudo apt install -y qemu-utils cloud-image-utils
```

#### Load the NBD module

```
sudo modprobe nbd max_part=8
```

#### Create a folder to hold the file system

```
sudo mkdir imageroot
```

#### Connect the NBD device to the image - use fdisk to list its partitions and mount the image root to imageroot:

```
sudo qemu-nbd --connect=/dev/nbd0 resolute-server-cloudimg-amd64.img
sudo fdisk /dev/nbd0 -l
sudo mount /dev/nbd0p1 imageroot
```

#### Dump the root fs contents to be restore later

```
cd imageroot/
sudo tar czf ../root-fs-contents.tgz *
cd ..
```

#### Unmount the rootfs to create the LVM resources

```
sudo umount imageroot 
```

#### Create the LVM resources:

```
sudo pvcreate -y /dev/nbd0p1
sudo vgcreate imgvg0 /dev/nbd0p1
sudo lvcreate -l 100%FREE -n root imgvg0
```

#### Create the ext4 partition - notice the -L to label it so fstab still works. 

```
sudo mkfs.ext4 -L cloudimg-rootfs /dev/imgvg0/root
```

#### Mount the empty partition and extract the original contents into it

```
sudo mount /dev/imgvg0/root imageroot/
cd imageroot/
sudo tar xf ../root-fs-contents.tgz 
cd ..
```

#### Mount the boot and efi partitions, as well as the special partitions. We need this to update grub correctly

```
sudo mount /dev/nbd0p13 imageroot/boot/
sudo mount /dev/nbd0p15 imageroot/boot/efi
for i in /dev /dev/pts /proc /sys /run; do sudo mount -B $i imageroot$i; done
```

#### chroot to imageroot and update grub

```
sudo chroot imageroot/
update-grub
exit
```

#### Unmount everything and disconnect the NBD device

```
for i in /dev/pts /dev /proc /sys /run; do sudo umount imageroot$i; done
sudo umount imageroot/boot/efi 
sudo umount imageroot/boot
sudo umount imageroot
sudo qemu-nbd --disconnect /dev/nbd0
```

#### Create a VM using the new image

```
cp resolute-server-cloudimg-amd64.img change2lvm-vda.qcow2

cat > user-data <<EOF
#cloud-config
password: passw0rd
chpasswd: { expire: False }
hostname: change2lvm
version: 2
EOF

cloud-localds -d qcow2 change2lvm-seed.qcow2 user-data

virt-install --boot uefi --import --osinfo detect=on,require=off --name change2lvm --memory 8192 --vcpus 4 --disk=change2lvm-vda.qcow2,bus=virtio --disk=change2lvm-seed.qcow2,bus=sata --network network=default,model=virtio --boot hd --noautoconsole

virsh console change2lvm
```

#### ASCIINEMA

[![asciicast](https://asciinema.org/a/oAYsiXbrm7DDNDgg.svg)](https://asciinema.org/a/oAYsiXbrm7DDNDgg)