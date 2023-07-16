# Homescripts README

This is my configuration that works for me the best for my use case of an all in one media server. This is not meant to be a tutorial by any means as it does require some knowledge to get setup. I'm happy to help out as best as I can and welcome any updates/fixes/pulls to make it better and more helpful to others.

I use the latest rclone stable version downloaded direclty via the [script install](https://rclone.org/install/#script-installation) as package managers are frequently out of date and not maintained. I see no need to use docker for my rclone setup as it's a single binary and easier to maintain and use without being in a docker. 

[Change Log](https://github.com/animosity22/homescripts/blob/master/Changes.MD)

## Home Configuration

- Verizon Gigabit Fios
- Dropbox with encrypted media folder
- Linux
- Intel(R) Core(TM) i7-7700 CPU @ 3.60GHz
- 32 GB of Memory
- 1TB - root/system disk
- 2TB SSD for rclone disk cache
- Spinning disk for big storage pools

I adjusted my mounts on my Linux machine to use BTRFS over EXT4/XFS and found a huge performance improvement.

/etc/fstab

```bash
/dev/disk/by-uuid/f12cd4cf-d9e5-4022-8d16-5ccde5c4273e / btrfs defaults 0 1
/dev/disk/by-uuid/7B20-481C /boot/efi vfat defaults 0 1

# SSD
UUID=7f93b2af-ad87-4db1-aa82-682136cec07a /cache auto defaults 0 0
UUID=e065dcaa-e548-45e6-a226-f5ea83b5ab22 /data auto defaults 0 0
```

## Dropbox

I migrated away from Google Drive to Dropbox as there still is an Enterprise Standard plan that seems to be unlimited space but I disliked
the upload and download limits so I made the change. Dropbox is similiar to API usage compared to Google but there is not a pacer by default
I work around that by setting a limit for transactions per second on the API via my mount command. API usage in Dropbox is tied to each
application registration so I seperate out my apps and use one for uploading, one for movies and one for television shows as to never
overload a particular one and allow easy reporting in the console.

## My Workflow

The design goals for the workflow are to limit the amount of applications that being used and limit the amount of scripts that are being used as part of the process. That was the reason to remove mergerfs and the upload script from the workflow. This does remove the ability to use hard links in the process but the trade off of having duplicated files for a short period outweighed the con.

Worfklow Pattern:
1. Sonarr/Radarr identify a file to be downloaded
2. qBit/NZBget downloads a file to local spinning disk (/data)
3. Sonarr/Radarr see the download is complete, file is copied from spinning disk (/data) to the respective rclone mount (/media/Movies or media/TV)
4. Rclone waits the delay time (1 hour in this setup) and uploads the file to the remote

This workflow has a lot less moving parts and reduces the amount of things that can break. There is a local cache drive for rclone (/cache) that is used for the vfs-cache-mode full that stores the uploads before they get uploaded and any downloaded cache files. The only breakpoint here is if the cache area gets full, but generally that should not happen as files are uploaded within an hour and it is a 2TB SSD in this setup that should offer plenty of space. If that disk is too small, there can be issue with it filling up and creating an issue. Disk is cheap enough though that should not be a problem.


### Installation

My Linux setup:

```bash
lsb_release -a
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 22.04 LTS
Release:	22.04
Codename:	jammy
```

Fuse needs to be installed for a rclone mount to function. `allow-other` is for my use and not recommended for shared servers as it allows any user to see a rclone mount. I am on a dedicated server that only I use so that is why I uncomment it.

You need to make the change to /etc/fuse.conf to `allow_other` by uncommenting the last line or removing the # from the last line.

```bash
sudo vi /etc/fuse.conf
root@gemini:~# cat /etc/fuse.conf
# /etc/fuse.conf - Configuration file for Filesystem in Userspace (FUSE)

# Set the maximum number of FUSE mounts allowed to non-root users.
# The default is 1000.
#mount_max = 1000

# Allow non-root users to specify the allow_other or allow_root mount options.
user_allow_other

```

Two rclone mounts are used and this could be expanded if there was a need for multiple mount points. The goal here is each mount point for unique for each "pair" of applications uses. As an example, Sonarr/Plex use the /media/TV mount and point to that specifically. This allows for downloads and uploads to work on a mount point. Any uploads are handled by rclone without the need for an additional upload script. The upload delay is configurable and 1 hour is the parameter being used here.

```bash
/media/Movies (rclone mount with vfs cache mode full)
/media/TV (rclone mount with vfs cache mode full)
```

My `rclone.conf` has an entry multiple entries for Dropbox as unlike Google that does rate limiting per user, Dropbox does it per application that is registered so there is an application registration for each mount point to break up the API hits. As of this time, the sweet spot for Dropbox is 12 TPS per application registration so each of my mounts takes that into account. The same encryption password is used for both in my case for ease of use and that is not a requirement.

My rclone looks like: [rclone.conf](https://github.com/animosity22/homescripts/blob/master/rclone.conf)

They are all mounted via systemd scripts.

My media starts up items in order:

1) [rclone-movies service](https://github.com/animosity22/homescripts/blob/master/systemd/rclone-movies.service) This is a standard rclone mount, the post execution command allows for the caching of the file structure in a single systemd file that simplies the process.
2) [rclone-tv service](https://github.com/animosity22/homescripts/blob/master/systemd/rclone-tv.service) This is a standard rclone mount, the post execution command allows for the caching of the file structure in a single systemd file that simplies the process.

### Docker

With the exception of rclone, all applications are setup in a docker-compose and leverage docker for ease of use, maintenance and upgrades. With Plex, it is advised to leverage a docker as for ensuring that hardware transcoding and HDR tone mapping support, only certain Linux OS flavors work easy without docker. By putting Plex in a docker, there is minimal configuration that needs to be done to get full hardware support as depending on hardware, it does get complex.

[Plex HDR Tone Mapping Support](https://support.plex.tv/articles/hdr-to-sdr-tone-mapping/)

Docker install for each operating system can be instructions are here: [Docker Install Ubuntu](https://docs.docker.com/engine/install/ubuntu/)

The docker-compose.yml below is what is being used for multiple applications as Sonarr, Radarr and Plex are included below. The key for hardware support is ensuring that /dev/dri is mapped and a single UID/GID is consistent in the configuration as UID=1000 and GID=1000 is the only user configured on my single server setup.

The docker setup is configured in /opt/docker and all the data for every application is stored in /opt/docker/data in this configuration. That is backed up on a daily basis to another location and occassinally to cloud storage depending on the risk appetite. 

#### My Docker-Compose

```bash
version: '3'
services:
  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    volumes:
      - /opt/docker/data/sonarr:/config
      - /media/TV:/media/TV
      - /data:/data
    ports:
      - 8989:8989
    restart: unless-stopped
      radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    volumes:
      - /opt/docker/data/radarr:/config
      - /media/Movies:/media/Movies
      - /data:/data
    ports:
      - 7878:7878
    restart: unless-stopped
      plex:
    image: lscr.io/linuxserver/plex:latest
    container_name: plex
    network_mode: host
    devices:
     - /dev/dri:/dev/dri
    privileged: true
    environment:
      - PUID=1000
      - PGID=1000
      - VERSION=docker
      - TZ=America/New_York
    volumes:
      - /opt/docker/data/plex:/config
      - /media/Movies:/media/Movies
      - /media/TV:/media/TV
    restart: unless-stopped
```

The override below forces the docker server to require the rclone mounts and if the rclone mounts stop, systemd will stop all the docker services that are running. This allow the dependencies to be done to ensure that the applications do not lose media and stop if an issue with rclone occurs.

```bash
gemini:/etc/systemd/system/docker.service.d # cat override.conf
[Unit]
After=rclone-movies.service rclone-tv.service
Requires=rclone-movies.service rclone-tv.service
```

I use docker compose for all my serivces and have portainer there for easier looking at things when I don't want to connect to a console. I use the same user ID/groups for my docker to simpify permissions. My plex compose is basic and looks like:

```bash
  plex:
    image: lscr.io/linuxserver/plex
    container_name: plex
    network_mode: host
    devices:
     - /dev/dri:/dev/dri
    privileged: true
    environment:
      - PUID=1000
      - PGID=1000
      - VERSION=docker
      - TZ=America/New_York
    volumes:
      - /opt/docker/data/plex:/config
      - /media/Movies:/media/Movies
      - /media/TV:/media/TV
    restart: unless-stopped
```

/dev/dri is a must for hardware transocding.

## Plex Tweaks

This is a legacy tweak for the plexmediaserver service to get around it running as the plex user and require services be running. This allows me to keep my trash empty on as if mount has a problem, it will stop plex. This requires /var/lib/plexmediaserver (on Linux) to be changed to the owner and group defined below.

```bash
gemini: /etc/systemd/system/plexmediaserver.service.d # cat override.conf
[Unit]
After=rclone-movies.service rclone-tv.service
Requires=rclone-movies.service rclone-tv.service

[Service]
User=felix
Group=felix
```

These tips and more for Linux can be found at the [Plex Forum Linux Tips](https://forums.plex.tv/t/linux-tips/276247)

### Plex

- `Enable Thumbnail previews` - off: This creates a full read of the file to generate the preview and is set per library that is setup
- `Perform extensive media analysis during maintenance` - off: This is listed under Scheduled Tasks and does a full download of files and is ony used for bandwidth analysis when streaming.

### Sonarr/Radarr

- `Analyze video files` - off: This also fully downloads files to perform analysis and should be turned off as this happens frequently on library refreshes if left on.

## Caddy Proxy Server

I use Caddy to server majority of my things as I plug directly into GitHub oAuth2 for authentication. I can toggle CDN on and off via the proxy in the DNS.

My configuration is [here](https://github.com/animosity22/homescripts/blob/master/PROXY.MD).
