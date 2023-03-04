# Smokeping Guide

This guide will walk you through how to set up smokeping on a raspberry pi using docker and ubuntu server.

## Install Ubuntu Server

### 1. Download Ubuntu Image

In our example we're using a pi 3, so we'll go and grab their Ubuntu install image from [here](https://ubuntu.com/download/raspberry-pi). When choosing the image, I recommend using 64 bit.

### 2. Install Etcher

You can use any variety of programs for this, including something as basic as ```dd```, however for ease of use I recommend using [etcher](https://www.balena.io/etcher/). Once downloaded, install the program.

### 3. Write image

Once etcher is installed, unzip the the Ubuntu Server image and use etcher to write it to an SD card. It will prompt you to type in a password, this is normal.

_If the SD card was already mounted it may fail, just retry and it should work fine._

### 4. Edit pre-boot configuration files

Once etcher has finished, your OS will automatically mount the boot partition of the SD card ready for use (if it doesn't, simply remove and reconnect your SD card from your machine). Ubuntu server ships with two files in the boot partition to allow customisations such as network config and passwords before first boot:

[network-config](config_files/network-config)<br>
[user-data](config_files/user-data)

_These config files are only used on the first boot, modifying them after first boot will have no effect. If you want to re-apply you must re-flash the SD card or manually edit the pi's configuration files by logging in to it._

#### network-config

The [network-config](config_files/network-config) set up the network interfaces for options such as:

* DHCP
* Static IP
* Wifi Networks

For the purposes of this tutorial it is safe to leave it as standard - this will give the pi a dynamic DHCP address.

#### user-data

The [user-data](config_files/user-data) file is used for a variety of configuration items, such as setting up users, installing packages, configuring hostnames etc. For the purposes of this tutorial we are going to do the following things:

* Disable automatic password expiry for the default user
* set the hostname to _smokepi_

To do this, edit the ```user-data``` config file, find the following section and modify it from:

```
chpasswd:
  expire: false
  list:
  - ubuntu:ubuntu

```

To:

```
chpasswd:
  expire: true
  list:
  - ubuntu:ubuntu

hostname: smokepi
```

Optionally you could change the default user from _ubuntu_ to _adam_ and the default password from _ubuntu_ to _password1_ by changing it to this instead:

```
chpasswd:
  expire: true
  list:
  - adam:password1

hostname: smokepi
```

### 5. First boot

Once the pi has booted, we must wait for it to stop installing updates before we can continue. To check this, run the following command:

```bash
while true; do date; ps aux | grep unattended; sleep 5; done
```

This will start a never-ending command that will print all processes matching "unattended" back to the console. When updates are complete, it should simply return two lines that match.

### 6. Install Docker

This section was taken from docker's own documentation available [here](https://docs.docker.com/engine/install/ubuntu/).

#### Install pre-requisites

First we must install the packages that Ubuntu needs in order to configure the docker repository:

```bash
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```

#### Add Docker GPG key

Next we must install Docker's GPG key so that your pi trusts the docker packages it is about to install:

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

#### Add Docker repositories

Now we will install the docker packages. If you installed 64 bit Ubuntu (recommended), run:
```bash
sudo add-apt-repository \
   "deb [arch=arm64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

If you installed 32 bit Ubuntu, run:
```bash
sudo add-apt-repository \
   "deb [arch=armhf] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

#### Install Docker packages

Now we must install docker itself with the following commands:

```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-compose
```

#### Add ubuntu user to docker groups

Now we must add the _ubuntu_ user to the _docker_ group. Doing this ensures that you can run the `docker` command without prefixing it with `sudo`:

```bash
usermod -aG docker ubuntu
```

Once this is done you will need to disconnect and reconnect your ssh connection in order for your user's group membership to be picked up.

_If you changed the username in the [user-data](#user-data), change `ubuntu` for whichever username you chose._

#### Test that docker is working

Finally you should check that docker is up and running by running `docker ps`. Below is what the output should look like when running the command:

```bash
ubuntu@smokepi:~/ $ docker ps
CONTAINER ID        IMAGE                   COMMAND                  CREATED             STATUS              PORTS                  NAMES
```

### 7. Set up mDNS

This step is optional, however it ensures that the smokeping will be available at [http://smokepi.local](http://smokepi.local) once we're finished installing smokeping, so it is advisable. Install the following packages:

```bash
sudo apt install avahi-daemon libnss-mdns mdns-scan
```

## Install smokeping

We're going to set up smokeping by using [linuxserver's smokeping container on docker hub](https://hub.docker.com/r/linuxserver/smokeping).

### 1. Create smokeping user

We are going to create a smokeping user so that the container can store it's files in a folder it has permissions to:

```bash
useradd smokeping
```

### 2. Create folder structure

We're going to create a few directories in `/opt/` for smokeping to store it's data in:

```bash
mkdir -p /opt/smokeping/{data,config,compose}
```

Verify that all folders are there with `ls /opt/smokeping`, it should look something like this:

```bash
ubuntu@smokepi:~/ $ ls /opt/smokeping
compose  config  data
```

### 3. Change ownership of folders

Now we will change the ownership of the folders `config` and `data` so that they're owned by the smokeping user & group:

```bash
sudo chown smokeping:smokeping /opt/smokeping/{config,data}
```

### 4. Create docker-compose file

Now we will create the file [docker-compose.yaml](config_files/docker-compose.yaml). Before we do we need to find the ID of the smokeping user, so that we can update the docker-compose file with this information. run `id smokeping` to find out smokeping's user ID:

```bash
ubuntu@smokepi:~/ $ id smokeping
uid=1001(smokeping) gid=1001(smokeping) groups=1001(smokeping)
```

You can see from the above output that the smokeping user & group ID is 1001. Using this information, we will open an editor with the command `sudo nano /opt/smokeping/compose/docker-compose.yaml` and paste the following content:

```yaml
---
version: "2.1"
services:
  smokeping:
    image: linuxserver/smokeping
    container_name: smokeping
    environment:
      - PUID=1001
      - PGID=1001
      - TZ=Europe/London
    volumes:
      - /opt/smokeping/config:/config
      - /opt/smokeping/data:/data
    ports:
      - 80:80
```

Ensuring that `PUID` & `PGID` variables under the `environment` section are updated to match the output of the command `id smokeping` we ran earlier.


### 5. Start smokeping

Now we are ready to start smokeping:

```bash
docker-compose -f /opt/smokeping/compose/docker-compose.yaml -d
```

Once the command has completed you can confirm it's running using ```docker ps```, the output should look something like as follows:

```bash
ubuntu@smokepi:~$ docker ps
CONTAINER ID        IMAGE                   COMMAND             CREATED             STATUS              PORTS                NAMES
1ee540a613a2        linuxserver/smokeping   "/init"             2 hours ago         Up 2 hours          0.0.0.0:80->80/tcp   smokeping
```

Congratulations! You should now be able to access your smokeping installation via a browser at [http://smokepi.local](http://smokepi.local). Smokeping should stay up until you tell it to stop, and should survive the pi being restarted.

## Smokeping Configuration & Maintenance

### Starting, stopping, restarting, destroying:

The following commands will allow you to start, stop, restart or destroy smokeping:

```bash
# Start smokeping
docker-compose -f /opt/smokeping/compose/docker-compose.yaml up -d
# Stop smokeping
docker-compose -f /opt/smokeping/compose/docker-compose.yaml stop
# Restart smokeping
docker-compose -f /opt/smokeping/compose/docker-compose.yaml restart
# Destroy smokeping
docker-compose -f /opt/smokeping/compose/docker-compose.yaml down
```
_Even the destroy command will not remove data in /opt/smokeping, so all the above commands are safe to run_.

### Configuration files

Smokeping has now stored a copy of it's configuration files in `/opt/smokeping/config`, on a default install the folder contents should look something like this:

```bash
ubuntu@smokepi:~$ ls /opt/smokeping/config/
Alerts  Database  General  Presentation  Probes  Slaves  Targets  httpd.conf  pathnames  site-confs  ssmtp.conf
```

_After editing any of the below files, be sure to restart smokeping for the changes to take effect_.

#### /opt/smokeping/config/Probes

The `Probes` file defines the commands used to monitor hosts. Here's mine as an example:

```
*** Probes ***

+ FPing

binary = /usr/sbin/fping

+ Curl

binary = /usr/bin/curl
agent = User-Agent: Mozilla/5.0 (X11; Fedora; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.116 Safari/537.36
extraargs = -s
urlformat = http://%host%/
follow_redirects = yes
include_redirects = yes
require_zero_status = yes
pings = 4
```

The `Fping` section is standard, however I have made some modifications to the `Curl` section that I recommend adopting so that you don't end up connecting to sites to often and being marked as a robot (e.g. the _pings_ setting has been reduced from a default 20 to 4). I have also set a `agent` string to a desktop string for the same reason.

#### /opt/smokeping/config/Targets

The `Targets` file defines the menu layout and hosts you will be monitoring.

##### HTTP
For instance I added a new menu item called "Custom HTTP" and added the bbc news website in my configuration file as so:

```
+ Custom_http
probe = Curl
menu = Custom http
title = Custom http

++ bbcnews
menu = BBC News
title = BBC News
host = bbcnews.co.uk
```

Note that because I specified `Curl` as the probe under Custom_http, all child entries below it have inherited this setting. This means that, rather than smokeping pinging bbcnews.co.uk with an icmp packet, it instead will try and connect to the web server and report back whether or not it was successful.

##### Standard ping

`Fping` is the standard probe and so, if you don't define a probe at any level, it will always ping with icmp. Below is an example with a menu item "DNS" with a google dns server as a host, that uses ping by default:

```
+ DNS
menu = DNS
title = DNS

++ GoogleDNS1
menu = Google DNS 1
title = Google DNS 8.8.8.8
host = 8.8.8.8
```

### Clear graphs

To clear graph data, simply clear data folder like so:

```
sudo rm -rf /opt/smokeping/data/*
```
