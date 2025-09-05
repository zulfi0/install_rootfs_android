# install linux rootfs android

## install linux rootfs android

#### Requirement

* Rooted android devices.
* Android 12+ (tested on 13)

Note:

```
tested on device redmi note 8 with android 13 (custom rom)
```

#### Download linux rootfs

Download linux rootfs:

*   Parrot

    https://deb.parrot.sh/parrot/iso/5.3/

    http://mirror.math.princeton.edu/pub/parrot/iso/5.3/Parrot-rootfs-5.3\_arm64.tar.xz (48 M)
*   Kali

    https://kali.download/nethunter-images/current/rootfs/kalifs-arm64-full.tar.xz (full: 1.7 GB)

    https://kali.download/nethunter-images/current/rootfs/kalifs-arm64-minimal.tar.xz (minimal: 135 MB)
*   Ubuntu

    https://cdimage.ubuntu.com/ubuntu-base/releases/22.04/release/

Create directory at `/data/local/rootfs/parrot-rootfs` (parrot-rootfs is the name of linux rootfs downloaded).

```bash
mkdir -p /data/local/rootfs/parrot-rootfs
```

Extract the rootfs under /data/local/rootfs/parrot-rootfs

```bash
cd /data/local/rootfs/parrot-rootfs/
busybox tar -xpf <path to linux rootfs> --numeric-owner
```

create a boot-script.sh

```bash
#!/system/bin/sh
# Android 10 and older don’t have binderfs. use /dev instead.

# Rootfs path (adjust this)
ROOTFS="/data/local/rootfs/kali-arm64"

# Make sure rootfs exists
if [ ! -d "$ROOTFS" ]; then
    echo "[-] Rootfs not found at $ROOTFS"
    exit 1
fi

echo "[*] Setting up mounts for chroot..."

# Bind mount rootfs onto itself (needed by some tools like Docker)
busybox mount -o bind $ROOTFS $ROOTFS

# Fix setuid issue on /data
busybox mount -o remount,dev,suid /data

# Standard mounts
busybox mount --bind /dev $ROOTFS/dev
busybox mount --bind /sys $ROOTFS/sys
busybox mount --bind /proc $ROOTFS/proc
busybox mount -t devpts devpts $ROOTFS/dev/pts

# Cgroups (needed for some services)
busybox mount -t tmpfs -o mode=755 tmpfs $ROOTFS/sys/fs/cgroup
mkdir -p $ROOTFS/sys/fs/cgroup/devices
busybox mount -t cgroup -o devices cgroup $ROOTFS/sys/fs/cgroup/devices

# --- Binder setup ---
if [ -d /dev/binderfs ]; then
    echo "[*] Binderfs detected, mounting into chroot..."
    mkdir -p $ROOTFS/dev/binderfs
    busybox mount -r -o bind /dev/binderfs $ROOTFS/dev/binderfs
fi

# Mount sdcard
if [ -d /sdcard ]; then
    mkdir -p $ROOTFS/sdcard
    busybox mount --bind /sdcard $ROOTFS/sdcard
fi

# --- External SD card detection ---
echo "[*] Checking for external SD..."

# Look for any directory under /storage except "emulated" and "self"
for dir in /storage/*; do
    base=$(basename "$dir")
    if [ "$base" != "emulated" ] && [ "$base" != "self" ]; then
        if [ -d "$dir" ]; then
            echo "[+] Found external storage at $dir"
            mkdir -p $ROOTFS/external_sd
            busybox mount --bind "$dir" $ROOTFS/external_sd
            MOUNTED_EXTERNAL_SD=1
            break
        fi
    fi
done


# Enter chroot
echo "[*] Entering chroot..."
if [ -e "$ROOTFS/bin/sudo" ]; then
    busybox chroot $ROOTFS /bin/sudo su
else
    busybox chroot $ROOTFS /bin/su -
fi

# Cleanup on exit
echo "[*] Cleaning up mounts..."
if [ -d /dev/binderfs ]; then
    busybox umount -l $ROOTFS/dev/binderfs
fi

busybox umount $ROOTFS/dev/pts
busybox umount -l $ROOTFS/dev
busybox umount $ROOTFS/sys/fs/cgroup/devices
busybox umount $ROOTFS/sys/fs/cgroup
busybox umount $ROOTFS/sys
busybox umount $ROOTFS/proc
busybox umount $ROOTFS/sdcard 2>/dev/null

# Unmount external_sd if it was mounted
if [ "$MOUNTED_EXTERNAL_SD" = "1" ]; then
    busybox umount $ROOTFS/external_sd
fi

echo "[+] Done."
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

#### SSH

run ssh server in chroot: install openssh

```bash
sudo apt install openssh-client openssh-server
```

change root password.

```bash
passwd root
```

add the following line to the `/etc/ssh/sshd_config`

```bash
PermitRootLogin yes
```

then start the service

```bash
mkdir -p /run/sshd
service ssh start
```

or

```bash
mkdir -p /run/sshd
/usr/sbin/sshd -D &
```

#### Troubelshoots

if encountered `Download is performed unsandboxed as root` warning run the following inside the rootfs:

```bash
groupadd -g 3003 aid_inet
groupadd -g 3004 aid_net_raw
groupadd -g 1003 aid_graphics
usermod -g 3003 -G 3003,3004 -a _apt
usermod -G 3003 -a root
```

**Audio & VNC ?**

follow the following steps: https://ivonblog.com/en-us/posts/termux-chroot-ubuntu/

##

Reference:

https://ivonblog.com/en-us/posts/termux-chroot-ubuntu/

##

What is the different with the tutorial in reference?

Nothing major. I just realized that in the reference link they didn’t unmount /sdcard. Also, instead of using `su -`, I use `sudo su`, because when I run tmux my bash/zsh prompt breaks with `su -`, but works fine with `sudo su`.
