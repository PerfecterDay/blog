# rootfs
{docsify-updated}

rootfs（根文件系统）就是 Linux 启动后作为 / 的那棵目录树，里面包含 /bin、/etc、/lib、/sbin 等用户空间程序和配置；它可以来自 ext4 等磁盘文件系统，也可以来自 initramfs。

如果 rootfs 位于硬盘/SSD/Flash 等块设备上，那么内核必须能够识别底层的文件系统类型（例如 ext4、xfs、squashfs），才能把它 mount 成根文件系统 /。但 rootfs 本身不是一种文件系统类型，它描述的是“作为系统根目录 / 的文件系统”。rootfs 也可以来自 initramfs，这时就直接存储在内存中，甚至可以是 tmpfs，因此不一定存储在硬盘上。

```
BIOS/UEFI
   ↓
Bootloader
   ↓
Linux Kernel
   ↓
内核 rootfs
   ↓
initramfs
   ↓
/init
   ↓
找到真正 rootfs
   ↓
switch_root
   ↓
/sbin/init
   ↓
systemd (Ubuntu)
   ↓
PID 1
   ↓
用户空间
```