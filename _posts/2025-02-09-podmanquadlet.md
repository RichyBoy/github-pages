---
title: "Quadlet, podlet and SystemD"
date: 2025-02-09
layout: post
tags: blog linux development podman quadlet podlet systemd
---

## What am I trying to achieve?

I currently have four containers running various tools: 
- Container registry
- PostgresQL server
- Nginx reverse proxy 
- Django webserver 

These containers are running under Podman on my OpenSuSE Tumbleweed microserver. I want these containers to spin up on system start-up, in user space, and SystemD is the tool for the job. This isn't the actual end-state as I want to be able to spin up onto other hosts and actually have meaningful pods, but that will be a different post for a different day.

Podman a number of years ago integrated the Quadlet project into its toolchain and I'm not going to describe it when the manual page says so much [https://docs.podman.io/en/stable/markdown/podman-systemd.unit.5.html](https://docs.podman.io/en/stable/markdown/podman-systemd.unit.5.html) but the first sentence is key: "Podman supports building, and starting containers (and creating volumes) via systemd by using a systemd generator". What this means in practice: create something that is a SystemD service file in a certain directory, and this shall be known as a Podman Quadlet. Add all the extra Podman stuff you need to this Quadlet file, using the same format that SystemD files follow. When SystemD re-scans for configs it will eventually invoke a generator to create/overwrite the unit file. That's a bit of a loose description but should help clarify things.

To be exact here's the SystemD generators installed on my system:
```
richard@microserver:~> ls -ltar /usr/lib/systemd/user-generators/
total 36
lrwxrwxrwx 1 root root    31 Jan 22 06:22 podman-user-generator -> ../../../libexec/podman/quadlet
-rwxr-xr-x 1 root root 30920 Jan 31 15:27 systemd-xdg-autostart-generator
drwxr-xr-x 1 root root   104 Feb  6 13:17 .
drwxr-xr-x 1 root root  2410 Feb  6 13:17 ..
```

## Introducing podlet

I have only written incredibly simple SystemD service files and that's the way I like it. A Quadlet file would mean having to write a service file and then populate it with podman stuff - so there is a chance that:
- Won't get the service part of the quadlet correct due to ignorance
- Won't get the podman part of the quadlet correct due to ignorance

On the other hand, you can in principle throw away the command line and compose files and maintain it entirely through the quadlet - if your "ignorance" has been sufficently satisifed. I am not so sure that this is great for developing a container service in the first place where you probably do want to fiddle around a lot, it's an artifact of a "deployed" container IMO. 

There is a tool called **podlet** and the source code lives here [https://github.com/containers/podlet](https://github.com/containers/podlet). To summarise the important parts: "Podlet generates Podman Quadlet files from a Podman command, compose file, or existing object." and "Podlet is primarily a tool for helping to get started with Podman systemd units, aka Quadlet files. It is not meant to be an end-all solution for creating and maintaining Quadlet files". It details several ways of installing it but on SuSE just a regular "zypper in podlet":
```
zypper search -f podlet
Loading repository data...
Reading installed packages...

S  | Name   | Summary                  | Type
---+--------+--------------------------+--------
i+ | podlet | Podman quadlet generator | package
```
Not that I recall installling it but i+ means it was a user request so I must have done :)

All of sudden the ignorance part can go out of the window, yay \o/

## A reasonable example of some quadlets
I previously blogged about my postgres setup. The docker-compose file can be found here: [https://github.com/RichyBoy/general/tree/main/postgres-configs](https://github.com/RichyBoy/general/tree/main/postgres-configs) 
```
podlet --install compose docker-compose.yml
```
This is the output from the command:
```
richard@microserver:/data/dev/postgres_setup> podlet --install compose docker-compose.yml 
# database.container
[Container]
ContainerName=postgres_test
Environment=POSTGRES_PASSWORD_FILE=/run/secrets/pgsqladmin PGDATA=/var/lib/postgresql/data/testing_times
EnvironmentFile=.env
HealthCmd=pg_isready -d postgres
HealthInterval=10s
HealthRetries=5
HealthTimeout=5s
Image=richard/postgres:latest
Network=postgres-network
PublishPort=5432:5432
Secret=pgsqladmin
UserNS=keep-id:uid=999,gid=999
Volume=pgdata.volume:/var/lib/postgresql/data
Volume=/data/dev/postgres_setup/init-user-db.sh:/docker-entrypoint-initdb.d/init-user-db.sh

[Install]
WantedBy=default.target

---

# postgres-network.network
[Network]
Driver=bridge
IPv6=true

[Install]
WantedBy=default.target

---

# pgdata.volume
[Volume]
Device=/data/dev/volumes/postgres_data
Driver=local
Options=bind
Type=none

[Install]
WantedBy=default.target
```

This is great! It's not written any configs other than to stdout but by inspection you can see a container, a network and a volume. Three files can now be created from that output, and the contents should be placed into one of these locations, as per the manual page:
```
Quadlet files for non-root users can be placed in the following directories

    $XDG_RUNTIME_DIR/containers/systemd/

    $XDG_CONFIG_HOME/containers/systemd/ or ~/.config/containers/systemd/

    /etc/containers/systemd/users/$(UID)

    /etc/containers/systemd/users/
```
As mentioned, I'm interesting in user space, E.g. **systemctl --user** and thus all of my configs live in **~/.config/containers/systemd**

As the podlet documentation cautions it's a very good starting place. Consider though that my postgres **docker-compose.yml** specifies a network and a volume to use - but they weren't created in that compose file, infact the network section is missing a subnet if creating one from scratch. So in fact podlet has done what it said on the tin, so you must be mindful of little things like that. The full config for all my quadlets can be found here: [https://github.com/RichyBoy/general/tree/main/quadlets](https://github.com/RichyBoy/general/tree/main/quadlets)

The container registry was invoked from a command line and this is what it looked like with podlet:

```
podlet --install --description "Richard's OCI registry" podman run -d   --restart=always  \
  --name registry   --replace   --net registry_net   -v "$(pwd)"/cert:/certs   \ 
  -v /data/dev/registry:/var/lib/registry   -e REGISTRY_HTTP_ADDR=0.0.0.0:443   \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/local.crt   -e REGISTRY_HTTP_TLS_KEY=/certs/local.decrypted.key \
  -p 5001:443   registry:2
```
So again, be careful to check thoroughly what the network/volume sections say. 

## Issuing SystemD commands

I've seen several articles that needlessly want you to set certain variables.. and there is no need, but the two you see in particular are:
- **DBUS_SESSION_BUS_ADDRESS**
- **XDG_RUNTIME_DIR**

These are set by **pam_systemd** upon login so unless you've configured something special this would all be set already. However:
```
loginctl enable-linger user
```
This is most definitely required if you want your user space podman services to actually run headless - it enables user services to remain active without the user being logged in.

To make SystemD consider your quadlets:
```
systemctl --user daemon-reload
```
To inspect the full user service list:
```
systemctl list-unit-files --type=service --user
```
From this point the volume and the network services just need to be ran the once, with a **systemctl --user start postgres-network-network.service** etc.

Looking specifically at my postgres network service:

![Image](/assets/images/quadlet_1.png "Naming mis-match") 

and then looking at what podman thinks it has after I've ran everything up:

![Image](/assets/images/quadlet_2.png "Naming mis-match")

The naming convention isn't actually ideal as the actual podman network is going to be named after the SystemD name - maybe there is a way to control this but I was OK with editing the quadlet to use the actual name. 

![Image](/assets/images/systemctl_1.png "Inspecting a SystemD service") 

the above details one of the actual running services, and the log at the bottom can now be inpected via journalctl or by podman logs, the former may be appropriate if you have some telemetry scraping going on.

The final thing to do once you've got it all running:

```
richard@microserver:~/.config/containers/systemd> podman ps
CONTAINER ID  IMAGE                                             COMMAND               CREATED     STATUS               PORTS                                        NAMES
ab8e065a1647  microserver.lan:5001/foofoo:latest                runserver [::]:80...  2 days ago  Up 2 days            0.0.0.0:4321->8000/tcp, 4321/tcp             foofoo
964f5d31ba15  microserver.lan:5001/richard/reverseproxy:latest  nginx -g daemon o...  2 days ago  Up 2 days            0.0.0.0:4040->80/tcp, 0.0.0.0:4443->443/tcp  webdev-rp
6924a4e8111f  docker.io/library/registry:2                      /etc/docker/regis...  2 days ago  Up 2 days            0.0.0.0:5001->443/tcp, 5000/tcp              registry
2ef71deea519  microserver.lan:5001/richard/postgres:latest      postgres              2 days ago  Up 2 days (healthy)  0.0.0.0:5432->5432/tcp                       postgres_test
```

## Final remarks
It does what it says on the tin. This does require careful consideration and inspection of quadlet files - but there are still several things left to do:
- A service for creating firewall rules, if truly automating a deploy of everything required. 
- Apropos above that may be within the realms of a proper orchestration tool, so far, this is all about service configuration for containers with SystemD
- I haven't considered pods. foofoo, my Django project, needs to be running before Nginx and should both be in the same pod.
- Apropos above the nginx container restarts within existing timing/retry contraints. SystemD order dependency needs to be specified, that is, do it properly!
