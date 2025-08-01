# rootfs初始化
1. 以fedora最新版的系统描述,安装debootstrap,通过deboostrap获取ubuntu的最小根文件系统
```
sudo debootstrap --arch=amd64 jammy . http://archive.ubuntu.com/ubuntu
//可根据需要设置不同架构和版本的
```

2. 得到rootfs之后自定义
假定第一步获取到的rootfs的位置在/mnt/rootfs
```
sudo mount --bind /dev /mnt/rootfs/dev
sudo mount --bind /proc /mnt/rootfs/proc
sudo mount --bind /sys /mnt/rootfs/sys

sudo chroot /mtn/rootfs /bin/bash
#可以设置国内的下载源
apt update
apt install vim net-tools iproute2 systemd ifupdown

#添加用户
adduser test
passwd test

#设置主机名称和hosts

echo "test" > /etc/hostname
echo "127.0.0.1 localhost" /etc/hosts

apt install openssh-server
systemctl enable ssh

#interface,属于ifupdown的配置,自动配置虚拟机网络
# 修改/etc/network/interface如下:
---------------------------------
source /etc/network/interfaces.d/*


auto lo
iface lo inet loopback


auto ens3//我这边的是ens3
iface ens3 inet dhcp
---------------------------------

# 清理与退出
exit //离开当前的chroot环境
sudo umount /mnt/rootfs/dev
sudo umount /mnt/rootfs/proc
sudo umount /mnt/rootfs/sys
```
# 将自定义的roofs打包成ext4的镜像
```
dd if=/dev/zero of=rootfs.img.ext4 bs=1M count=2048
mkfs.ext4 rootfs.img.ext4
mkdir temp_rootfs
sudo mount rootfs.img.ext4 temp_rootfs
sudo cp -a /mnt/rootfs/* temp_rootfs
sudo umount temp_rootfs
```
# 主机侧网络配置
```
# 设置基于bridge+tap+nat的方式
# 借用了boxes的virtbr0及其内置的NAT和DHCP,所以可以简化很多
sudo ip tuntap add dev tap_owl mode tap user $(whoami)
sudo ip link set tap_owl up
sudo brctl addif virtbr0 tap_owl
# 其中virtbr0是主机侧已启动的,我平时会使用boxes配置的虚拟机
```

# qemu运行
```
qemu-system-x86_64 \
  -kernel bzImage -m 8192M\
  -netdev tap,id=net0,ifname=tap_owl,script=no,downscript=no\
  -device virtio-net-pci,netdev=net0\
  -drive file=rootfs.image.ext4,format=raw,if=virtio \
  -append "root=/dev/vda rw crashkernel=768M console=ttyS0 " \
  -drive file=dump_disk.qcow2,format=qcow2,if=virtio\
  -serial mon:stdio\
  -enable-kvm

# netdev指定了ifname为tap_owl
```
