---
title: "Synology Gluetun Setup"
date: 2025-01-18T12:00:00-08:00
draft: true
tags = ["infra", "synology", "vpn", "nas"]
---

## Introduction
I have a Synology NAS that I use as my own local file storage system. It's a pretty capable little box that I also use to host a few docker containers that run inside of a VPN.

I wanted it to be fully dockerized and have had to go through a few different iterations to get the setup in a spot that feels just right. This blog is going to detail that setup. The hope here is that it helps someone else out in the future (as well as helps myself out as it's a sane place to document the work I've done on it thus far.)

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
```

You can also start it too if you want ahead of time if you haven't already created the tun.

```console
sudo systemctl start tun.service
```
From here on out, whenever your NAS boots, you'll have the tun device automatically enabled! Woohoo!

## Adding a container to the VPN

Once we have glutetun set up and running, we can add a container to it by updating our docker compose file.

```yaml
# ... gluetun config
  qbittorrent:
    image: linuxserver/qbittorrent:5.0.2
    container_name: qbittorrent
    environment:
      - PUID=1026
      - PGID=100
      - TZ=America/Los_Angeles
      - UMASK_SET=022
      - WEBUI_PORT=8090
      - TORRENTING_PORT=33796
    volumes:
      - /volume1/docker/qbt/config:/config # will vary based on your system
      - /volume1/media/downloads/seeding:/downloads # will very based on your system
    network_mode: service:gluetun # important piece here. Tells the container to route all of its traffic through the glutetun vpn
    restart: unless-stopped
    depends_on:
      - gluetun
```
The `network_mode` here is the important piece. This mode works a little bit differently than simlpy bringing the container online and having it live in the same _docker network_. They actually are configured to run on the same internal network, or rather, on the same local host. 

Normally when two containers come online in the same network, they are routable via their container or service names. In this respect, they share the same local host (and must not have overlapping binding ports).

This is the magical step that ensures gluetun has a stranglehold over the internet traffic of any of the other containers running inside of the network.

Another important note here is that if you want to expose qbittorrent's web UI for use, the port binding must actually live on the `gluetun` service definition itself, _not_ on the qbittorrent definition.

```yaml
gluetun:
  # .. config
  ports:
    - 8090:8090 # qbittorrent port
  environment:
    - FIREWALL_VPN_INPUT_PORTS=33796
```
Another note for qbittorrent is that you will need to expose the firewall vpn input port on gluetun to be qbit's torrenting port.

## Fixing occasional network loss

Based on your VPN setup, Gluetun will occasionally need to switch VPN servers. This is pretty common if you've set up your VPN provider using Wireguard and set it to a pretty wide region like an entire country. When this happens, the ip tables will change and the port forward will stop working.

This plagued my for quite some time as I'd check on my container and it wouldn't be connectable.
Fortunately, this is pretty easy to fix.

```sh
#!/bin/sh

output=$(docker exec qbittorrent sh -c 'netstat -nlp | grep -q 10.167.*.*'; echo $?)
if [ "$output" -ne 0 ]; then
    docker restart qbittorrent
fi

exit 0
```

What this script effectively does is check the container for the IP address that is expected by your Wireguard container. If it doesn't find it in the netstat output, the container is simply restarted.
This is a quick and easy script that can be put behind a cronjob to ensure that your torrenting container is set up and connectable.
Personally I have a cron set up to run this script every 10 minutes or so, which keeps the torrent container nice and healthy.

## Conclusion
With this setup, you should have longevity in your container network setup and need to do very little maintenance on it. Enjoy and happy tunneling!