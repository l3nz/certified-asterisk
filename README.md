# Certified Asterisk Docker Image (unofficial)
Docker image for Certified Asterisk 13 (unofficial package). Maintained by AVOXI. Primary purpose is for use during [AstriCon 2015](http://astricon.net) presentation.

## Running Asterisk
The following sets of commands will show you how to use the Asterisk image along with some other supporting volume containers to get going.

***

### Using Docker Compose
You can skip over starting everything up manually by using [Docker Compose](https://docs.docker.com/compose/) but there are still a few gotchas that we've been running into.

To start, you require the development version of Docker Compose, slated to become version 1.5.0. Without that you don't get access to port ranges in the `docker-compose.yml` file, and a couple other changes to starting Docker volume containers in detach mode.

With that being said, you can approximate the start up that is documented in the manual start up documentation below, which might be good enough for your local environment.

#### Start up the data volumes
Unfortunately you can't do a simple `docker-compose up` because you won't really get access to the Asterisk console, which means you can't run Asterisk CLI commands, and you can't drop to the shell to figure out what your IP address is once you start up the container (if using DHCP and pipework as described later in the document, you might be able to look at the reservation cache on your DHCP service).

First we start up the volumes for Asterisk:

```sh
docker-compose up -d spool moh sounds-en
```

A this point you have all the volumes spun up in detached mode.

#### Starting Asterisk
Next step is to start up Asterisk. The `docker-compose.yml` file will understand what volumes to attach to on start up, so this is pretty straight forward.

> **NOTE**: this is one of the gotchas I ran into with Compose. You can't start Asterisk in a detached state, or you never get the console to come up correctly. This also means you can't use `^P^Q` to detach. You might just be better off to start up Asterisk via the manual process, but if you're just playing around, maybe this is less of an issue.

Start up Asterisk with the following command:

```sh
docker-compose run -d --service-ports asterisk
```

This will result in an Asterisk prompt, and you'll be able to run a `docker attach` at this point as well.

Next, skip down to the section on **Network Considerations** and running `pipework`.

***

### Starting Containers Manually

#### Starting up the data volumes

First we'll start up a few Docker Volume Containers (DVC). These are pre-built volume containers that host the sound prompts and music on hold files for Asterisk.

> The sound prompts in the example are English prompts, both core and extras, with formats `G.729`, `SLIN`, and `WAV`.
> Files in the MOH volume are same formats, using the opsound files from Digium.

##### Start Docker Volume Containers
```sh
docker run -d --name="asterisk-sounds-en" docker.io/avoxi/asterisk-sounds-en:latest
docker run -d --name="asterisk-moh" docker.io/avoxi/asterisk-moh:latest
```

##### Create volume containers for Asterisk data
```sh
docker run -v /var/spool/asterisk \
           --name="asterisk-spool" centos:7 \
           sh -c 'echo Asterisk Spool Volume'

```

#### Starting Asterisk and Mounting Volumes
```sh
docker run -ti \
           --volumes-from=asterisk-sounds-en:ro \
           --volumes-from=asterisk-moh:ro \
           --volumes-from=asterisk-spool \
           -v `pwd`/etc-asterisk:/etc/asterisk \
           docker.io/avoxi/certified-asterisk:latest
```

***

## Networking Considerations
There are various methods of setting up networking with Docker, but for our purposes, it is easier to have the container to get a real IP address from our DHCP server, and then expose the ports.

In general this is not the right way to do it, but for our testing purposes, it does get us an Asterisk container up and running in a short period of time, and allows us to connect a couple of phones to it.

In order to get going in short order we can use [pipework](https://github.com/jpetazzo/pipework) to attach our container to our local LAN and request and IP address from our local DHCP server.

The following command assumes that you've passed a `--name=ast01` value to the `docker run` command. If you didn't then just pass the hash of the container that is running that you want to use dhclient for networking. The below example also assumes your LAN interface is `enp8s0`.

```sh
docker run -ti --name="ast01" \
           --hostname="ast01" --add-host="ast01:127.0.0.1" --net=none \
           --volumes-from=asterisk-sounds-en:ro \
           --volumes-from=asterisk-moh:ro \
           --volumes-from=asterisk-spool \
           --expose=5060 \
           --expose=10000-20000 \
           -v `pwd`/etc-asterisk:/etc/asterisk \
           docker.io/avoxi/certified-asterisk:latest

# detach from the container with ^P^Q

sudo pipework enp8s0 ast01 dhclient
```

Now if you attach to the running container, you will have an IP address from your local LAN.

### Hostname Gotcha
You are likely to see a message if you don't setup the `/etc/hosts` file to have the containers hostname configured:
```
getaddrinfo("896f1da008b0", "(null)", ...): No address associated with hostname
Unable to lookup '896f1da008b0'
```
We specify a hostname using the `--hostname` (`/etc/hostname`) and local hostname resolution using the `--add-host` (`/etc/hosts`) flags. Without this the `/etc/hostname` will simply be the container ID.

You can still resolve this by adding the container ID (`896f1da008b0`) to the `/etc/hosts` file on the system.

From the Asterisk `*CLI>` run:
```
*CLI> !vi /etc/hosts
```

None of this should be necessary though if you use the flags as indicated.
