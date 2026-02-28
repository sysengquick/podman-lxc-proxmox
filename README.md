# Rootless Podman on LXC on Proxmox

## Background

This repo contains files and directions for getting rootless podman running on an LXC container on a proxmox VE host.
A lot of this information should be credited to users on the [Proxmox Forum](https://forum.proxmox.com/threads/podman-in-rootless-mode-on-lxc-container.141790/post-661678).

Video guide is available on [YouTube](https://www.youtube.com/playlist?list=PLvadQtO-ihXsyM0eVcrw4FBx0UmZmj0az).

## Directions

### Create LXC Container for Podman

1. Download debian-13-standard proxmox container template.
2. Create LXC container from template.
   1. Hostname `podman`.
   2. Check `Unprivileged container` and `Nesting`.
   3. Add SSH key if you want to login as root. Debian disallows password root logins over ssh by default.
   4. Storage, CPU, Memory can be left at the defaults. These can be easily changed later.

### Configure the LXC Container

Login to the new container as root.

1. Update packages.

    ```sh
    apt update && apt upgrade -y
    ```

2. Install podman.

    ```sh
    # podman-docker and docker-compose are optional, but I find them useful
    apt install -y podman podman-docker docker-compose
    ```

3. Switch editor from nano to vim (optional -- if you like nano, use nano)

    ```sh
    apt install -y vim

    # pick vim from the options presented
    update-alternatives --config editor
    select-editor
    ```

4. Add non-root user for container

    ```sh
    apt install -y sudo

    # change NEW_USER to whatever you want to call the user
    NEW_USER=podman
    useradd -m -G sudo -m /bin/bash $NEW_USER
    passwd $NEW_USER
    ```

5. Shut down LXC container to edit configuration.

    ```sh
    # alternatively: you can use shutdown from the proxmox GUI menu
    poweroff
    ```

### Edit LXC Configuration on Proxmox Host

Some options are not available in the GUI.
Login to the proxmox host, either as root, or a user with sudo privileges.
Find the PVE ID of your LXC container in the GUI (it's the number preceding the name).

1. Edit the LXC container configuration.

    Example files can be found in the /etc/pve/lxc directory.

    ```sh
    PVE_LXC_ID=100
    sudoedit /etc/pve/lxc/${PVE_LXC_ID}.conf
    ```

    See [original](etc/pve/lxc/1-original.conf) and [podman](etc/pve/lxc/2-podman.conf) for examples.

    Add the following lines to the end of the config.

    ```text
    lxc.idmap: u 0 100000 200000
    lxc.idmap: g 0 100000 200000
    lxc.cgroup2.devices.allow: c 10:200 rwm
    lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
    ```

    You can map your non-root proxmox user to your non-root lxc user.
    You can also map your graphics card for proxmox or jellyfin.
    [Examples](etc/pve/lxc/3-samba_and_plex.conf)

    If you change the non-root user ID in your container, you'll have to fix the permissions on the proxmox host.
    My LXC filesystem is at `/rpool/data/subvol-100-disk-0`, but it may be different for you.
    Find the home directory for the user and sudo chown -R 1000:1000 to fix it.

2. Edit the PVE subuid/subgid ranges.

    Example files can be found in the /etc/subuid directory.

    ```sh
    sudoedit /etc/subuid
    sudoedit /etc/subgid
    ```

    The root user should look like this:

    ```text
    root:100000:200000
    ```

    If there are other entries for non-root users, they will need to be removed or adjusted.

    See [original](etc/subuid/1-original) and [podman](etc/subuid/2-podman).

    This has the potential to break things for non-root users using subuid/subgids on the pve host.
    Hopefully, that was just the shadow-utils package allocating those at creation and they aren't being used.
    If not, make sure you understand the implications of changing them.
    This guide cannot help with that.

3. Restart the LXC container.

### Running non-root user quadlets on lxc

1. Login to the podman lxc container.
2. Enable linger.

    ```sh
    # change podman to the non-root username
    sudo loginctl enable-linger podman
    ```

3. Allow non-root user to bind low ports &lt; 1024

    ```sh
    echo 'net.ipv4.ip_unprivileged_port = 53' | sudo tee /etc/sysctl.d/99-ports.conf
    sudo sysctl -p /etc/sysctl.d/99-ports.conf
    ```

4. Disable podman-user-wait-network-online.service

    ```sh
    systemctl --user edit podman-user-wait-network-online.service
    ```

    Add the following lines in between the comment blocks in the editor.

    ```systemd
    [Service]
    ExecStart=
    ExecStart=/bin/true
    ```

5. Create user quadlets directory

    ```sh
    QUADLET_DIR=$HOME/.config/containers/systemd
    mkdir -p $QUADLET_DIR
    # put whatever .container file you want in the directory
    cp dnsmasq.container $QUADLET_DIR
    # ! NO SUDO HERE FOR THESE COMMANDS !
    systemctl --user daemon-reload
    systemctl --user start dnsmasq
    ```

### Enable Hardware Video Transcoding

GPU accelerated transcoding requires some additional steps on the proxmox server.

#### Additional Drivers

You may need additional drivers to be able to use all the features of your video card.
I have an Intel Arc A310.
AMD or nVidia cards may need different drivers, but I can't help with that.

1. Enable the non-free debian repos

    ```sh
    sudoedit /etc/apt/sources.list.d/debian.sources
    ```

    On the lines that say "Components:", make sure `non-free` is there.
    `non-free-firmware` is NOT the same thing.

2. Install the intel-media-va non-free drivers.

    ```sh
    sudo apt update && sudo apt install -y intel-media-va-driver-non-free
    ```

#### Add Device Permissions

1. Install the acl package.

    ```sh
    sudo apt install -y acl
    ```

2. Add udev rules.

    ```sh
    sudoedit /etc/udev/rules.d/99-video.rules
    ```

    Add the following to the file

    ```text
    SUBSYSTEM=="drm", KERNEL=="renderD*", MODE="0660", RUN+="/usr/bin/setfacl -m g:200910:rw /dev/dri/%k"
    ```

    Group 200910 is the same as group 911 in the rootless podman plex container inside lxc.
    If you run your plex server as a different user, you will need to adjust the group ID.

    For non-root users inside the podman container, you can calculate the proxmox group ID.
    Take the GID inside the container, add 200000 and subtract 1.
    GID 911 in podman = 911 + 200000 - 1 = 200910 on proxmox.

    For the root user inside the podman container, the UID/GID on the lxc container are the same as the rootless user.
    If you mapped that rootless user into lxc from proxmox (as I did for samba), then it's the same on proxmox.
    If you did not map the user from proxmox, then it's 100000 + UID/GID, e.g. 101000 for the default user.

3. Reload udev rules

    ```sh
    sudo udevadm control --reload-rules
    sudo udevadm trigger --subsystem-match=drm
    getfacl /dev/dri/renderD128
    ```

    Make sure the acl list shows your new group rw permissions.

    ```text
    # file: dev/dri/renderD128
    # owner: root
    # group: render
    user::rw-
    group::rw-
    group:200910:rw-
    mask::rw-
    other::---
    ```
