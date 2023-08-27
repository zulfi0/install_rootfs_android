# install linux rootfs android
Install linux rootfs directly in android rooted devices

## Requirement
- Rooted android devices.
- Android 12+ (tested on 13)

Note:
```text
tested on device redmi note 8 with android 13 (custom rom)
```
## Download linux rootfs
Download linux rootfs:
- Parrot

  http://mirror.math.princeton.edu/pub/parrot/iso/5.3/Parrot-rootfs-5.3_arm64.tar.xz (48 M)
- Kali

  https://kali.download/nethunter-images/current/rootfs/kalifs-arm64-full.tar.xz (full: 1.7 GB)
  
  https://kali.download/nethunter-images/current/rootfs/kalifs-arm64-minimal.tar.xz (minimal: 135 MB)

- Ubuntu
  
  https://cdimage.ubuntu.com/ubuntu-base/releases/22.04/release/

 Create directory at `/data/local/rootfs/parrot-rootfs` (parrot-rootfs is the name of linux rootfs downloaded).
 ```bash
mkdir -p /data/local/rootfs/parrot-rootfs
```

Extract the rootfs under /data/local/rootfs/parrot-rootfs
```bash
cd /data/local/rootfs/parrot-rootfs/
busybox tar -xpf <path to linux's rootfs> --numeric-owner
```

create a boot-script.sh
```bash
#!/bin/sh

# The path of rootfs
ROOTFS="/data/local/rootfs/parrot-rootfs"

# Fix setuid issue
busybox mount -o remount,dev,suid /data

busybox mount --bind /dev $ROOTFS/dev
busybox mount --bind /sys $ROOTFS/sys
busybox mount --bind /proc $ROOTFS/proc
busybox mount -t devpts devpts $ROOTFS/dev/pts

# Mount sdcard
busybox mount --bind /sdcard $ROOTFS/sdcard

# chroot into rootfs
# use `sudo` instead of `su` to fix the tmux prompt.

if [ -e "$ROOTFS/bin/sudo" ]; then
        busybox chroot $ROOTFS /bin/sudo su
else
        busybox chroot $ROOTFS /bin/su -
fi

# Umount everything after exit
busybox umount $ROOTFS/dev/pts
busybox umount $ROOTFS/dev
busybox umount $ROOTFS/proc
busybox umount $ROOTFS/sys
busybox umount $ROOTFS/sdcard
```
Make the script executable
```bash
chmod +x boot-script.sh
```

start the script
```bash
sh boot-script.sh
```

or if using termux
```bash
su -c ./boot-script.sh
```

Fix resolver
```bash
echo 'nameserver 1.1.1.1' > /etc/resolv.conf
echo 'nameserver 8.8.8.8' >> /etc/resolv.conf
```

## Troubelshoots
if encountered `Download is performed unsandboxed as root` warning run the following inside the rootfs:
```bash
groupadd -g 3003 aid_inet
groupadd -g 3004 aid_net_raw
groupadd -g 1003 aid_graphics
usermod -g 3003 -G 3003,3004 -a _apt
usermod -G 3003 -a root
```

### Audio & VNC ?
follow the following steps:
https://ivonblog.com/en-us/posts/termux-chroot-ubuntu/

#
Reference:
https://ivonblog.com/en-us/posts/termux-chroot-ubuntu/

#

What is the different with the tutorial in reference?


Nothing, i just relize in the reference link did not unmount /sdcard and also instead of using `su - root`
i use `sudo su` because when i use `tmux` my bash/zsh prompt is broken but with `sudo su` didn't

