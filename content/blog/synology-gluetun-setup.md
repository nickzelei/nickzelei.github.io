---
title: "Synology Gluetun Setup"
date: 2025-01-08T20:51:37-08:00
draft: true
tags = ["infra", "synology", "vpn", "nas"]
---

## Introduction
I have a Synology NAS that I use as my own local file storage system. It's a pretty capable little box that I also use to host a few docker containers that run inside of a VPN.

I wanted it to be fully dockerized and have had to go through a few different iterations to get the setup in a spot that feels just right. This blog is going to detail that setup. The hope here is that it helps someone else out in the future (as well as helps myself out as it's a sane place to document the owork I've done on it thus far.)

At the end of this, you'll have setup a fully working VPN docker as well as a qbittorrent (or any other) container running all traffic through it.

## Requirements and Setup

There are a few requirements needed before getting started.

1. Have Docker setup and installed on the NAS
2. Have SSH setup and be SSH'd into the box
3. A VPN that is either OpenVPN or Wireguard that works with Gluetun

## Setting up Gluetun

The first thing to get set up is the [Gluetun](https://github.com/qdm12/gluetun) VPN container.
Gluetun is a VPN client in a Docker container written in Go. It supports OpenVPN, Wireguard, and a slew of other features. It runs totally in userland so Wireguard is not needed on the host machine. It's a neat tool and will be the core of this entire setup.

Let's see what a basic docker compose file will look like for a gluetun setup.
I'm essentially going to paste in a simplified version of what I currently have and explain the necessary bits.

```yaml
version: "3"
services:
  gluetun:
    image: qmcgaw/glutetun
    container_name: gluetun
    environment:
      - PUID=1026
      - PGID=100
      - VPN_SERVICE_PROVIDER=<provider-name>
      - VPN_TYPE=wireguard
      - WIREGUARD_MTU=1320
      - WIREGUARD_PRIVATE_KEY=<KEY>
      - WIREGUARD_PRESHARED_KEY=<KEY>
      - WIREGUARD_ADDRESSES=<address>
      - SERVER_COUNTRIES=<countries>
      - TZ=America/Los_Angeles
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
    ports:
      # todo
    volumes:
      - /volume1/docker/gluetun:/gluetun
```
Now, most of this will depend on your specific setup and what VPN provider you are using, where you want your VPN to source from, etc. 

### Setting up `/dev/net/tun`
An important bit of info here is the `/dev/net/tun` device that is attached. This is typically provided via the Linux system, but is not installed on Synology boxes.
You can double check this by running `ls -l /dev/net/tun` to see if it exists.

This can be easily rectified by running the following command:

```console
/sbin/insmod /lib/modules/tun.ko
```
You'll find that the tun device now exists on your machine. However, this does not persist through servers boots!

This can be easily rectified however by setting up a `systemd` service that handles this for you on boot.

A service can be created pretty easily by adding a new file to the `/etc/systemd/service` directory. For myself, I've chosen it to be aptly named `tun.service`.

It looks like this:

```
[Unit]
Description=Run tun at startup
After=network.target

[Service]
Type=simple
User=root
ExecStart=/sbin/insmod /lib/modules/tun.ko
Restart=on-failure

[Install]
WantedBy=multi-user.target
``` 

You'll want to restart systemd to recognize the server, and then enable the service to start on boot:

```conosle
sudo systemctl daemon-reload

sudo systemctl enable tun.service.service
``

You can also start it too if you want ahead of time if you haven't already created the tun.

```console
sudo systemctl start tun.service
```
From here on out, whenever your Nas boots, you'll have the tun device automatically enabled! Woohoo.




