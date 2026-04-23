+++
title = "Building a fast, zero trust and secure Nextcloud server"
date = "2024-07-02T18:26:04+02:00"
author = ""
authorTwitter = "" #do not include @
cover = ""
tags = ["nextcloud", "rclone", "smb", "linux"]
keywords = ["", ""]
description = "How I build my nextcloud server infrastructure to be fast, secure and with no external trust."
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

## Why?

Using cloud services is great. You can have your data in one place and you don't have to copy it between devices. But online cloud providers have some drawbacks. Mainly privacy and vendor lock-in. Most online cloud providers have access to your data, as you may want to back up sensitive content, this is a no go for me. Also, you can only choose between the cloud storage sizes offered, with no alternative to switching to cheaper storage down the line without changing the entire cloud provider and its software stack.

> Currently no solution is sufficient for my needs.

## On site self hosting 

One of the, if not the, most popular solutions to this problem is to host your own cloud. The advantage of this is that your data stays on your device and you can upgrade your storage or hardware as you wish. But with that power comes a lot of responsibility: you have to keep your hardware running, update the software, and deal with slow home internet speeds. The upfront costs can also be quite high.

So self-hosting in your own home is less than ideal. What about using a VPS, i.e. a cloud computing provider? Here we have the problem that we don't have control over our data by default, as the cloud provider can technically inspect the data stored on their equipment.

> Self hosting is less than ideal most of the time. We want to use the cloud.

## How to do zero trust cloud computing? - defending against a malicous cloud provider

First, there are two ways the data could be seen by the cloud provider in plain text: The hard drive or the RAM. We need to protect both from any external source so that only we have access to the data.

> Encrypt RAM, harddrive and any external storage.

## Encrypting the harddrive

There is a great blog post that explains how to can enable [full disk encryption on a cloud server](https://community.hetzner.com/tutorials/install-ubuntu-2004-with-full-disk-encryption). This gives us the advantage that our data is encrypted at rest, but our server is running most of the time and needs to handle data as well as the encryption key somewhere in RAM. So how do we encrypt the RAM?


## Shifting trust to the silicon - AMD Secure Encrypted Virtualization (SEV)

Basically, AMD SEV exposes the chip's ability to encrypt the virtual machine's RAM independently for each VM. This also protects the key from the hypervisor - in our case the cloud provider - so we can compute without the cloud provider knowing what data resides have in RAM.

## Encrypt external storage

VPS storage can be quite expensive. Maybe you have some free cloud storage lying around from OneDrive or Google Drive. Or you might have bought a Hetzner storage box or some other external data storage. Most of them don't give you a way to run code to encrypt the storage yourself, for example. Wouldn't it be nice if we could somehow use this dumb storage to store our data and run our computing needs on a low storage VPS?


The first problem we need to solve is: how do we mount/connect to the storage? `rclone` is a fantastic tool that can interface with most major cloud storage providers, so we can use rclone to mount our storage. However, for my external storage it was much faster to use `mount.cifs` to mount the storage using the SMB protocol. The command I use:

```bash
mount.cifs -o user=username,pass=passw0rd //user.your-stroragedrive.com/folder /mnt/smb
```

Now we need to encrypt the storage on our encrypted RAM, harddrive VPS before sending it to storage.

rclone also provides a pipeline step to do just that: encrypt and decrypt storage requests using `rclone crypt`.

Another problem we need to solve is that we want to use the storage as if it were a hard drive directly attached to our machine, so that nextcloud thinks it is not an external drive, because I find nextcloud's handling of external storage lackluster, we can do better! So, to make our I/O operations on our drive 100% reliable, we'll use the virtual filesystem feature for `rclone mount` to buffer all I/O operations.

## Making all of it fast - building a cache hierachie.

We utilise different types of storage before our files and images are displayed by Nextloud. The storage hierarchy looks something like this:
> external storage --> NVMe SSD --> RAM

To cache external storage I/O operations, we use the rclone vfs cache feature, so we can have a large cache, say 10GB, that can cache our latest photos and videos every time we view our timeline. To cache the files needed for nextcloud, we use reddis to cache those files in RAM. The command I use to mount the rclone vfs cache is as follows:

```bash
rclone mount --daemon --vfs-cache-mode=full --allow-other --uid 33 --gid 33 --cache-dir /var/services/rclone/cache --vfs-cache-max-size 10G --vfs-read-ahead 32M --vfs-cache-max-age 72h --vfs-write-back 30s --dir-perms 0750 --file-perms 0644 storagebox_ENC: /var/services/rclone/decryptedStorage
```

I changed the file and directory permissions so that the `www-data` user for the Nextcloud server can access our files.

## Making the server secure - WireGuard

Personally, I don't trust Nextcloud or myself to keep up with all the latest emerging security threats (updating my server). So I don't want to expose my Nextcloud instance to the whole world wide web. Instead, I want to access it in an allow list fashion. So after enabling auto-update and only allowing public/private key authentication on my server, I also keep my Nextcloud instance behind a WireGuard VPN. Running all our traffic through WireGuard, as most tutorials on the web suggest, is a bit sub-optimal, as we only need to route the traffic to our server through WireGuard. This has the advantage that we can have our WireGuard connection open and always on, without the latency or speed penalty of using a VPN for all our traffic. So how do we do this? I will simply provide you with both the server and client configuration files for such a setup, as the WireGuard installation procedure is well documented online:

{{< code language="VIM" title="wg0.conf" id="1" expand="Show" collapse="Hide" isCollapsed="false" >}}
    [Interface]
    Address = 192.168.200.1/24 # the IP of the server
    ListenPort = 51820

    # Adjust your network interface. enp1s0 in my case
    PrivateKey = your_private_key_of_wireguard
    PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o enp1s0 -j MASQUERADE
    PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o enp1s0 -j MASQUERADE

    # each device needs its own private/public key pair!
    # computer
    [Peer]
    PublicKey = public_key_of_computer
    AllowedIPs = 192.168.200.2/32

    # laptop
    [Peer]
    PublicKey = public_key_of_laptop
    AllowedIPs = 192.168.200.3/32

    # phone
    [Peer]
    PublicKey = public_key_of_phone
    AllowedIPs = 192.168.200.4/32
{{< /code >}}


{{< code language="VIM" title="computer.conf" id="1" expand="Show" collapse="Hide" isCollapsed="false" >}}
    [Interface]
    PrivateKey = private_key_of_computer
    Address = 192.168.200.6/32

    [Peer]
    PublicKey = public_key_of_server
    AllowedIPs = 192.168.200.1/32 # this will route only traffic going to the server through WireGuard
    Endpoint = server_ip:51820
    PersistentKeepalive = 25
{{< /code >}}

Make sure you set your firewall to only allow inbound traffic to the SSH port and the WireGuard port, and block all HTTP/S traffic coming from the Internet. Only allow HTTP traffic coming from the WireGuard tunnel.

## Installing Nextcloud

I will only provide you with my `nextcloud.yml` file for my Docker Nextcloud instance as the installation process of Nextcloud is well documented online. Note that I installed aio-imaginary for faster preview image generation and ffmpeg in the Nextcloud image for hardware accelerated video encoding for streaming. But to be honest, these days I just stream my video files without re-encoding them first. The raw internet speed is just faster on my VPS rather than re-encoding the videos on the CPU.

{{< code language="yaml" title="nextcloud.yml" id="1" expand="Show" collapse="Hide" isCollapsed="false" >}}
version: "3"
services:

  nextcloud:
    image: mycloudy/nextcloud-ffmpeg
    build: .
    container_name: nextcloud
    restart: no
    volumes:
      - /var/services/nextcloud:/var/www/html
      - /var/services/rclone/decryptedStorage/nextCloudData:/data
    ports:
      - 192.168.200.1:80:80
    depends_on:
      - mysql8
      - redis
      - aio-imaginary

  mysql8:
    image: mysql:8.4
    container_name: mysql8
    volumes:
      - /var/services/mysql:/var/lib/mysql
    restart: no
    environment:
      MYSQL_ROOT_PASSWORD: your_mysql_password

  redis:
    image: valkey/valkey:8-alpine
    container_name: redis
    volumes:
      - /var/services/redis:/data
    restart: no

  aio-imaginary:
    image: nextcloud/aio-imaginary:latest
    container_name: imaginary
    cap_add:
      - SYS_NICE
    restart: no
    environment:
      - PORT=9000
    command: -concurrency 50 -enable-url-source -log-level debug
{{< /code >}}

## App recommendation
I recommend the Nextcloud app [memories](https://apps.nextcloud.com/apps/memories) as it mimics the likes of Apple Photos or Google Photos and is just so much faster and more feature rich than Nextcloud Photos. On iOS, you can also place a PWA link to memories on your homescreen and interact with the site without the Safari UI bars.

I would also recommend an alternative to the Nextcloud app's auto-upload for uploading images. Because the Nextcloud implementation is easily killed by iOS's limited background running and does not retry the next time it is opened. Also, the Nextcloud app on iOS is just horribly buggy. So I can recommend [PhotoSync](https://www.photosync-app.com/) for uploading, it can sync in the background and just works.