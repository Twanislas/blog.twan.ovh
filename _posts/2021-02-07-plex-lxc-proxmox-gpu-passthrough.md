---
title: "Plex in LXC on Proxmox, with Nvidia GPU passthrough"
categories:
  - Home server
tags:
  - Proxmox
---

Here's how I got my Plex server running inside an unprivileged CT on Proxmox, passing my mighty GTX 680 for transcodes. Bear in mind that this how _I_ did it. If you want to do the same, you'll have to adapt some commands and configs to your own setup.

## Considerations

- I'm running Proxmox 6.3 at the time of writing.
- My media files and Plex config files are stored on a ZFS pool, on the Proxmox host, all under UID/GID 1000.

## Configure Proxmox host

I try to avoid altering the original setup of bare metal hosts, but there's still some small adjustments needed.

### Backports repos

While I strongly discourage you to use backports in a prod setup, only the backported version of the Nvidia drivers worked for me.

Simply add the following line to your `/etc/apt/sources.list` :

```sh
deb http://deb.debian.org/debian buster-backports main contrib non-free
```

Don't worry, this will not update your existing packages when doing an APT upgrade, as the backports repo has a lower priority by default. However, packages installed from the backports will upgrade to their backported versions.

### Nvidia drivers

Here's a snippet from my bootstrap script for Proxmox. If you have an Nvidia GPU installed, it will install what you need to make use of it in containers.

The [Nvidia patch](https://github.com/keylase/nvidia-patch) removes the restriction of max 2 simultaneous encode/decode processes on consumer grade GPUs. You can estimate what your card is capable of (and thus adapt your plex settings) using [this calculator](https://www.elpamsoft.com/?p=Plex-Hardware-Transcoding). I've put it in a daily cron job but it would be better to hook into the driver install script as you'll have to run it on every driver update.

```sh
if lspci | grep -i vga | grep -i nvidia; then
  curl -s -L https://nvidia.github.io/nvidia-container-runtime/gpgkey | apt-key add -
  curl -s -L https://nvidia.github.io/nvidia-container-runtime/$(. /etc/os-release;echo $ID$VERSION_ID)/nvidia-container-runtime.list | tee /etc/apt/sources.list.d/nvidia-container-runtime.list
  apt update
  apt -y install pve-headers pkg-config
  apt -y install -t buster-backports nvidia-driver nvidia-smi nvidia-container-runtime nvidia-modprobe libnvidia-encode1 libnvcuvid1 libcuda1
  cat << EOF > /etc/cron.daily/nvidia-patch
  #!/bin/sh
  wget -q -O - https://raw.githubusercontent.com/keylase/nvidia-patch/master/patch.sh | bash -
  EOF
  chmod +x /etc/cron.daily/nvidia-patch
  /etc/cron.daily/nvidia-patch
  # reboot
fi
```

**Important:** Don't forget to reboot your host after the driver install.
{: .notice}

You can test if the install was successful by running `nvidia-smi`, it should show details about your GPU.

### Allow UID remapping by root in LXC

As I will use bind mounts to access my data and I'm using an [unpriviledged container](https://pve.proxmox.com/wiki/Unprivileged_LXC_containers), the UIDs and GIDs inside the container are remapped. However, I don't want that to happen for UID/GID 1000, as it's the one I use for my files and also the one I will use for the `plex` user.

This needs to be run on all the hosts that you plan to run your container on.

```sh
echo "root:1000:1" >> /etc/subuid 
echo "root:1000:1" >> /etc/subgid
```

## Create the LXC container

- Debian does __NOT__ work, use an Ubuntu template.

I used the UI to create my CT with the Ubuntu 20.04 template, because LTS. Nothing out of the usual stuff...

### Storage mountpoints

My media files are at `/tank/autopvr/data` and my Plex config is at `/tank/autopvr/config/plex`. I use the `noatime` mount option in a mere attempt to speed up IO a bit. I also disable backup and replication for those two.

Run this on the host, replacing `$CTID` with your container ID.

```sh
pct set $CTID -mp0 /tank/autopvr/config/plex,mp=/var/lib/plexmediaserver,mountoptions=noatime,replicate=0,backup=0
pct set $CTID -mp1 /tank/autopvr/data,mp=/data,mountoptions=noatime,replicate=0,backup=0
```

### Special LXC configs

Add this to your LXC container config file (found in `/etc/pve/lxc/<your container ID>.conf`)

```sh
lxc.idmap = u 0 100000 1000
lxc.idmap = g 0 100000 1000
lxc.idmap = u 1000 1000 1
lxc.idmap = g 1000 1000 1
lxc.idmap = u 1001 101001 64535
lxc.idmap = g 1001 101001 64535
lxc.hook.pre-start: sh -c '[ ! -f /dev/nvidia-uvm ] && /usr/bin/nvidia-modprobe -c0 -u'
lxc.environment: NVIDIA_VISIBLE_DEVICES=all
lxc.environment: NVIDIA_DRIVER_CAPABILITIES=all
lxc.hook.mount: /usr/share/lxc/hooks/nvidia
```

The `lxc.idmap` entries remove the UID/GID mapping in the unprivileged container for ID 1000, this allows me to run Plex as UID/GID 1000 and access files on my storage.

The `lxc.hook.pre-start` entry helps fix an issue where not all the Nvidia modules are loaded on in the kernel before the container is started.

The `lxc.environment` entries, you guessed it, pass env variables to the container.

The `lxc.hook.mount` entry runs a script provided but the Nvidia Container Runtime kit to allow passing of the GPU to the container.

More info about those configs can be found in `lxc.container.conf`'s [manpage](https://linuxcontainers.org/fr/lxc/manpages/man5/lxc.container.conf.5.html).

## Bootstrap the container and install Plex

```sh
timedatectl set-timezone Europe/Brussels
apt update
apt full-upgrade -y
apt install -y gnupg
echo deb https://downloads.plex.tv/repo/deb public main | tee /etc/apt/sources.list.d/plexmediaserver.list
wget -q -O - https://downloads.plex.tv/plex-keys/PlexSign.key | apt-key add -
apt update
apt install -y --force-confnew plexmediaserver
usermod -u 1000 plex
groupmod -g 1000 plex
```

Nothing very complicated here, update the container and install Plex. I changed the UID/GID of the Plex user to match the one used in my storage.

The great thing about LXC is that it's using the kernel from the host, thus its modules, and thus it can have access to the Nvidia driver running in the pve kernel. It means there's no need to install the Nvidia drivers in the container.

Like for the driver install on the host, you can test `nvidia-smi` inside your container and it should show details about your GPU.

There you go, that's it. You've got an unprivileged onctainer running Plex, and it uses your GPU for the transcoding. I hope this was useful :).

## Inspiration

- [Guide: Pass GPU (NVENC) into an LXC Container (with FFMPEG) -- Bradford's blog](https://bradford.la/2016/GPU-FFMPEG-in-LXC/)
- [Unprivileged LXC containers, -- Proxmox documentation](https://pve.proxmox.com/wiki/Unprivileged_LXC_containers)
- [Installation guide for PMS under Proxmox 5.1 within an LXC Container -- Linux Tips, Plex forums](https://forums.plex.tv/t/linux-tips/276247/15)
- [lxc.container.conf's manpage](https://linuxcontainers.org/fr/lxc/manpages/man5/lxc.container.conf.5.html)
