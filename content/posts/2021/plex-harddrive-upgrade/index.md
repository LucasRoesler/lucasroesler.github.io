---
title: "Upgrading and replacing my media hard drive for Plex"
date: 2021-01-31T00:00:00+02:00
tags:
  - plex
  - linux
  - systemd
  - fstab
draft: true
# coverImage: images/cover1.jpg
---

# Increasing my media drive

1. plugin drive
2. get the id using `blkid`
```sh
$ blkid  | grep '/dev/sd'
/dev/sda2: LABEL="Seagate Expansion Drive" BLOCK_SIZE="512" UUID="58B25E69B25E4C1E" TYPE="ntfs" PTTYPE="atari" PARTLABEL="Basic data partition" PARTUUID="f6851eb4-73ff-40b5-b572-70439ed3a49f"
```

We wan the UUID `58B25E69B25E4C1E`

3. edit my `/etc/fstab` so that my media mount uses the new drive's UUID
```sh
sudo vim /etc/fstab
```

My mount line looks like 

```fstab
UUID=58B25E69B25E4C1E	/disks/media ntfs-3g defaults,auto,rw,nofail,uid=1000,gid=1001 0 1
```

3. reload systemd 

if you do reload systemd, your disk will mount and then unmount automatically

```sh
/var/log/syslog:Jan 31 10:11:11 lucas-pi ntfs-3g[87591]: Mounted /dev/sda2 (Read-Write, label "Seagate Expansion Drive", NTFS 3.1)
/var/log/syslog:Jan 31 10:11:11 lucas-pi ntfs-3g[87591]: Mount options: nofail,allow_other,nonempty,relatime,rw,default_permissions,fsname=/dev/sda2,blkdev,blksize=4096
/var/log/syslog:Jan 31 10:11:11 lucas-pi systemd[1]: disks-media.mount: Unit is bound to inactive unit dev-disk-by\x2duuid-DE78DC0F78DBE3F5.device. Stopping, too.
/var/log/syslog:Jan 31 10:11:11 lucas-pi systemd[1]: Unmounting /disks/media...
/var/log/syslog:Jan 31 10:11:11 lucas-pi ntfs-3g[87591]: Unmounting /dev/sda2 (Seagate Expansion Drive)
/var/log/syslog:Jan 31 10:11:11 lucas-pi systemd[37731]: disks-media.mount: Succeeded.
/var/log/syslog:Jan 31 10:11:11 lucas-pi systemd[1]: disks-media.mount: Succeeded.
/var/log/syslog:Jan 31 10:11:11 lucas-pi systemd[1]: Unmounted /disks/media.
```

I found the solution here, https://mamchenkov.net/wordpress/2017/11/09/systemd-strikes-again-unit-var-whatever-mount-is-bound-to-inactive-unit/

and https://alex.mamchenkov.net/2017/11/09/mount-systemd/

> Apparently all records in /etc/fstab are converted to systemd units and all mounting is (these ugly days) done via systemd. So when I changed /etc/fstab, the systemd didn’t update the the related unit and was still trying to mount the old device.

4. finally mount the replacement drive

```sh
sudo mount /disks/media
```