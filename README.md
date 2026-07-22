# Install Linux Rootfs on Android

## Installing a Linux Rootfs on Android

#### Requirements

* A rooted Android device.
* Android 12+ (tested on Android 13).

Note:

```
Tested on a Redmi Note 8 running Android 13 (custom ROM).
```

#### Download a Linux Rootfs

Download one of the following:

*   Parrot

    https://deb.parrot.sh/parrot/iso/5.3/

    http://mirror.math.princeton.edu/pub/parrot/iso/5.3/Parrot-rootfs-5.3\_arm64.tar.xz (48 MB)
*   Kali

    https://kali.download/nethunter-images/current/rootfs/kalifs-arm64-full.tar.xz (full: 1.7 GB)

    https://kali.download/nethunter-images/current/rootfs/kalifs-arm64-minimal.tar.xz (minimal: 135 MB)
*   Ubuntu

    https://cdimage.ubuntu.com/ubuntu-base/releases/22.04/release/

#### Choosing Where to Store the Rootfs: Internal Storage vs. SD Card

You have two options for where the rootfs will live:

1.  **Directly on internal storage** (default) — skip ahead to [Create the Rootfs Directory](#create-the-rootfs-directory).
2.  **On an SD card, inside an image file** (recommended if you want to keep the rootfs on an SD card, since FAT32/exFAT don't support Linux file permissions or symlinks) — follow the steps below, extract the rootfs into the mounted folder, then continue from the boot-script section.

##### Storing the Rootfs on an SD Card

If you choose this option, you have to format the SD card as exFAT rather than FAT32.

> **Note:** "/data/local/nhsystem/kali-arm64" is the rootfs mountpoint used for this example, and `XXXX-XXXX` is the volume UUID/label of the SD card used, and it **will be different on each device**. Find the SD card's mount point first with:
> ```bash
> ls /mnt/media_rw/
> ```
> 
**1. Create the image file (10GB or more):**

```bash
cd /mnt/media_rw/XXX-XXXX
dd if=/dev/zero of=kali.img bs=1M count=10240
```

**2. Format it as ext4:**

```bash
mkfs.ext4 kali.img
```

**3. Mount the image:**

```bash
mount -o loop /mnt/media_rw/XXXX-XXXX/kali.img /data/local/nhsystem/kali-arm64
```

If you encounter error `mount: '/dev/block/loop0'->'/data/local/nhsystem/kali-arm64': Block device required`, use this workaround instead (note the addition of `loop0` to `loop1`):

```bash
losetup /dev/block/loop1 /mnt/media_rw/XXXX-XXXX/kali.img
mount -t ext4 /dev/block/loop1 /data/local/nhsystem/kali-arm64
```

Once the image is mounted, extract the rootfs into it as usual — see [Extract the Rootfs](#extract-the-rootfs) below.

When you're finished with your session, unmount it with:

```bash
umount -l /data/local/nhsystem/kali-arm64
```

**Resizing the image**:

If you need more space you can resize the image with the following method:

```bash
cd /mnt/media_rw/XXXX-XXXX/
umount -l /data/local/nhsystem/kali-arm64
dd if=/dev/zero bs=1M count=10240 >> /mnt/media_rw/XXXX-XXXX/kali.img
e2fsck -fy kali.img
resize2fs kali.img
```

#### Create the Rootfs Directory

Create a directory at `/data/local/rootfs/parrot-rootfs` (replace `parrot-rootfs` with the name of the rootfs you downloaded):

```bash
mkdir -p /data/local/rootfs/parrot-rootfs
```

#### Extract the Rootfs

Extract the rootfs archive into that directory:

```bash
cd /data/local/rootfs/parrot-rootfs
busybox tar -xpf <path to rootfs.tar.xz> --numeric-owner
```

#### Create the Boot Script

Create a `boot-script.sh` with the following contents:

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

Make the script executable:

```bash
chmod +x boot-script.sh
```

Run it:

```bash
sh boot-script.sh
```

Or, if you're using Termux:

```bash
su -c ./boot-script.sh
```

**Fix DNS resolution** (once inside the chroot):

```bash
echo 'nameserver 1.1.1.1' > /etc/resolv.conf
echo 'nameserver 8.8.8.8' >> /etc/resolv.conf
```

#### SSH

To run an SSH server inside the chroot, first install OpenSSH:

```bash
sudo apt install openssh-client openssh-server
```

Set a root password:

```bash
passwd root
```

Add the following line to `/etc/ssh/sshd_config`:

```bash
PermitRootLogin yes
```

Then start the service:

```bash
mkdir -p /run/sshd
service ssh start
```

Or, alternatively:

```bash
mkdir -p /run/sshd
/usr/sbin/sshd -D &
```

#### Troubleshooting

**"Download is performed unsandboxed as root" warning** — run the following inside the rootfs:

```bash
groupadd -g 3003 aid_inet
groupadd -g 3004 aid_net_raw
groupadd -g 1003 aid_graphics
usermod -g 3003 -G 3003,3004 -a _apt
usermod -G 3003 -a root
```

**Audio & VNC?**

Follow the steps here: https://ivonblog.com/en-us/posts/termux-chroot-ubuntu/

---

### Reference

https://ivonblog.com/en-us/posts/termux-chroot-ubuntu/

---

### How Does This Differ From the Referenced Tutorial?

Nothing major. A couple of small changes:

* The reference guide never unmounts `/sdcard` — this guide does.
* This guide uses `sudo su` instead of `su -`, because `su -` breaks the bash/zsh prompt when running tmux, while `sudo su` works fine.
