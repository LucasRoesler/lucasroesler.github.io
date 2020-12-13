---
title: "Moving the data directory of Multipass and Docker"
summary: "Quick walktrhough for how to move your Docker and Multipass data folders."
date: 2020-12-13T00:00:00+02:00
categories:
  - programming
  - multipass
  - docker
tags:
  - multipass
  - docker
keywords:
  - multipass
  - docker
comments: false
showPagination: false
autoThumbnailImage: true
draft: false
thumbnailImagePosition: left
coverSize: partial
coverImage: /images/library-898333_640.jpg
---

This weekend was my first adventure in [Snaps](https://snapcraft.io/). Unfortunately, this adventure quickly devolved into a murder mystery, for my hard drive. The first snap I try to compile triggers a "disk is almost out of space" notification?! Turns out building Snaps requires a surprising amount of disk space. But it wasn't entirely Multipass' fault,my good old friend Docker was eating up a decent chunk of my disk too.

If you are like me, you have a separate (larger) data partion and probably want the Docker and Multipass data directories to live there instead of your small OS root partition.

<!--more-->

# Getting my laptop
Earlier this year we got my wife a new computer. Originally, it was supposed to be a simple "web and netflix" machine. However, being American expats in Germany, the "market" had a different plan for us. Most budget computers in Germany _only_ come with German keyboard layouts. Very natural, no real complaint. But this means we had to buy something at the professional level. We choose a Dell XPS 13. 

The bad news is that my wife ultimately didn't like the laptop. Primarily, she didn't liek the keyboard. The good news is that it means _I_ got a new laptop this year. 

## Setting up my laptop
For the past 10 years I have been a Mac user due it being offered at work. But I have been itching to get back to Linux. So I installed Ubuntu 20.20 :)

In the past I have always used 2 or 3 partitions, one for the OS, one for data, and optionally one for a second OS. On the XPs I have a separate `/home` mount which contains almost all of my disk space 230GB and a root `/` partition with just 37GB.

I thought this would be enough space for the OS, until I tried to compile a snap. 5 mins later and a notification pops up "You are almost out of disk space, 1 GB remaining". Now this is shocking since I don't remember installing any massive pieces of software.

After several iterations of `df -h`, `du -h`, `lsof`, etc. I pin point the problem to: Docker and Multipass saving every image, vm, and data mount to my `/var` folder. After recovering my disk space, I set out to move the data locations for docker and multipass to my `/home` partition, which has plenty of space.

# Using the data partition

## Setting up docker
Docker is the easy one, it has a configuration specifically for this use case. 

1. First I cleaned up the existing Docker data, 

  ```sh
  docker system prune -f --volumes --all
  ```

  You can probably skip this, but I prefer fresh starts.

2. Then, stop the docker daemon:
  
  ```sh
  sudo service docker stop 
  ```

3. Next, create your new data folder

  ```sh
  sudo mkdir -p /home/docker/data
  ```
  the folder will be owned by root, but this is ok.

4. Now add or edit the configuration file `/etc/docker/daemon.json` so that it contains

  ```json
  { 
    "data-root": "/home/docker/data" 
  }
  ```

5. Now you can copy and remaining Docker metadata to the new location 

  ```sh
  sudo rsync -aP /var/lib/docker/ /home/docker/data
  ```

  If you are being cautious, rename the old data folder so that you can safely revert if anything goes wrong

  ```sh
  sudo mv /var/lib/docker /var/lib/docker.old
  ```

6. Resart the docker daemon and test everything is work

  ```sh
  sudo service docker start
  docker run --rm hello-world
  ```

7. Cleanup the old data

  ```sh
  sudo rm -rf /var/lib/docker.old
  ```

## Setting up multipass 
Multipass is a bit harder than Docker. It _can_ be done, but the feature is much newer. It was [just merged in October 2020](https://github.com/canonical/multipass/pull/1789)

The author [Michał](https://github.com/Saviq) has kindly (provided instructions)[https://github.com/canonical/multipass/pull/1789#issuecomment-704991752] that I have adapted and extended below

1. First I want to safely clear any cached multipass data, the simplest way to do this is the uninstall and reinstall multipass

  ```sh
  sudo snap remove multipass
  sudo snap install multipass
  ```

2. Make the new data directory

  ```sh
  sudo mkdir /home/multipass
  ```

3. I created a new group for my user and `root`, so that I would have simpler access to the data directory

  ```sh
  sudo groupadd multipass
  sudo usermod -a G root multipass
  sudo usermod -a -G multipass $USER
  sudo chown -R root:multipass /home/multipass
  ```

4. Stop you multipass daemon and enable the required volumn plugins for snap

  ```sh
  sudo snap stop multipass
  sudo snap connect multipass:removable-media  # for /mnt /media https://snapcraft.io/docs/removable-media-interface
  sudo snap connect multipass:all-home  # for /home/* https://snapcraft.io/docs/home-interface
  ```

5. Update the multipass service definition to set the required `MULTIPASS_STORAGE` environment variable.

```sh
sudo mkdir /etc/systemd/system/snap.multipass.multipassd.service.d/
sudo tee /etc/systemd/system/snap.multipass.multipassd.service.d/override.conf <<EOF
[Service]
Environment=MULTIPASS_STORAGE=/home/multipass
EOF
```

6. Reload the service defitions

  ```sh
  sudo  systemctl daemon-reload
  ```

7. Restart the multipass daemon

  ```sh
  sudo snap start multipass
  ```

8. Check that things are working

  ```sh
  $ ls -al /home/multipass
  total 16K
  drwxrwxr-x 4 root multipass 4,0K Dez 13 16:00 .
  drwxr-xr-x 6 root root      4,0K Dez 12 21:58 ..
  drwxr-xr-x 3 root root      4,0K Dez 13 16:00 cache
  drwxr-xr-x 6 root root      4,0K Dez 13 16:00 data
  ```

  If you start a new vm, e.g. using `multipass shell`, then you will
  start seeing data in the `/home/multipass/cache/vault/images`, for example.


## A networking hiccup

Doing all of that, I now have a workign Docker and (I thought) a working Multipass with images and other data stored in my data partition. But this is a premature victory. 

Within the last couple days, it seems that Docker changed the way it uses iptables. The consequence is that the `iptables` Multipass needs are disabled and the VMs have _no network access_.  See this [Github issue](https://github.com/canonical/multipass/issues/1866).

Networking questions are generally above my paygrade, but the summary seems to be that the Multipass snap uses legacy `iptables` while Docker switched to `nftables`. Once `nftables` are used, the legacy tables are ignored.

This looks like 

```sh
$ sudo iptables -S
# Warning: iptables-legacy tables present, use iptables-legacy to see them
-P INPUT ACCEPT
-P FORWARD DROP
-P OUTPUT ACCEPT
-N DOCKER
-N DOCKER-ISOLATION-STAGE-1
-N DOCKER-USER
-N DOCKER-ISOLATION-STAGE-2
-A FORWARD -j DOCKER-USER
-A FORWARD -j DOCKER-ISOLATION-STAGE-1
-A FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -o docker0 -j DOCKER
-A FORWARD -i docker0 ! -o docker0 -j ACCEPT
-A FORWARD -i docker0 -o docker0 -j ACCEPT
-A DOCKER-ISOLATION-STAGE-1 -i docker0 ! -o docker0 -j DOCKER-ISOLATION-STAGE-2
-A DOCKER-ISOLATION-STAGE-1 -j RETURN
-A DOCKER-USER -j RETURN
-A DOCKER-ISOLATION-STAGE-2 -o docker0 -j DROP
-A DOCKER-ISOLATION-STAGE-2 -j RETURN
```

The solution is to update the iptables/nftables. At this time, I am still researching how to do this without breaking my laptop.  If you already _know_ iptables and nftables, please let me know on [Twitter](https://twitter.com/TheAxeR)