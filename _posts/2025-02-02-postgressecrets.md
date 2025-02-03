---
title: "Configuring postgres with podman and podman secrets"
date: 2025-02-02
layout: post
tags: blog linux development podman registry postgres secrets userns_mode
---

## Databases in the SOHO environment

In may ways it would be simple just to run up an AWS or Azure instance to provide you a database server. I am not opposed to simple, far from it and I approve of simple, but my end goal is a role-out an entire app estate via Tofu (or helm/kubernetes) and a database is just one component of that vision.

As an experienced developer/engineer it is easy to tangle yourself up in decisions you would only make in the enterprise environment: I'm not running the storage on a NAS, I don't really care about performance and anyway it is going to run on my 2-core N54L microserver. That said, there are certain things that are sane if you are going to run a 'proper' database in a container: don't use /var storage, do setup proper credentials and where appropriate use secrets. As it happens my bind storage is on a RAID array.

## Example config can be found in my general repo

[https://github.com/RichyBoy/general/tree/main/postgres-configs](https://github.com/RichyBoy/general/tree/main/postgres-configs) . This location contains a number of files and I'll detail these one-by-one. Note, I later will post about quadlet and the compose file will vanish. I'm quite happy with SystemD starting and stopping my world and I'm super-happy to do this in user space, especially as I will being hosting live webservices on my public IPv6 address range at some point - the safer the better.

The **.env** file. I am using the default user and database, I've obscured the email but I don't specify the password here at all.
```
POSTGRES_USER=postgres
POSTGRES_DB=postgres
PGADMIN_DEFAULT_EMAIL=xxx@yyy.zzz
```

**Dockerfile** - if you actually read the postgres setup you will know that you will need to sort out your language codepage. Pretty simple to build, personally I signed it and pushed it to my local repo but that's just me.
```
FROM postgres:latest
RUN localedef -i en_GB -c -f UTF-8 -A /usr/share/locale/locale.alias en_GB.UTF-8
ENV LANG=en_GB.UTF-8
```

**init-user-db.sh** - the default postgres config doesn't include a root user for the database. It doesn't break anything by not including it, but it's good to know how to use the run-only-once-on-first-creation part of postgres works.
```
#!/bin/bash
createuser -s root
```

## Podman components - secrets, volumes and networks

Generate a secret with podman. There are several ways of doing it but in general you should avoid a shell "echo" as it can stay in a process list and shell history, OK a very small thing but we should do our best..

``` 
# create a file called ./secret.txt and add you password to it
podman secret create pgsqladmin ./secret.txt
```

Comamnd for the volume
```
podman volume create pgdata -o device=/data/dev/volumes/postgres_data -o type=bind
```

Command for the network. This is more interesting, as IPv4 means use a IPv4 network but enabling IPv6 means use both a IPV4 and IPv6 network. However the way to ensure IPv6 only (which is what I want) is to specify a subnet.
```
podman network create --subnet fd00:1:1::/64 postgres-network
```

These can all be inspected with their retrospective podman inspect commands but I'll show the secret in full:
```
-> podman secret inspect pgsqladmin
[
    {
        "ID": "76e1c8b09251a6a006c32df75",
        "CreatedAt": "2025-01-12T19:25:09.686194625Z",
        "UpdatedAt": "2025-01-12T19:25:09.686194625Z",
        "Spec": {
            "Name": "pgsqladmin",
            "Driver": {
                "Name": "file",
                "Options": {
                    "path": "/home/richard/.local/share/containers/storage/secrets/filedriver"
                }
            },
            "Labels": {}
        }
    }
]
```
This file specified doesn't store the password in plain text, and now you've created your secret don't forget to remove ./secret (or whatever you called it).

The network looks similar to this, this is just an excerpt and uses a different name from my docker-compose file, it's actually another network setup for when I moved to SystemD start/stop:
```
-> podman network inspect systemd-postgres-network
[
     {
          "name": "systemd-postgres-network",
          "id": "387cbdb247ad6adbbcbaba6cb080a77e781aa2d9a692ffb83fb31da078fd43e5",
          "driver": "bridge",
          "network_interface": "podman1",
          "created": "2025-01-29T12:43:51.800477685Z",
          "subnets": [
               {
                    "subnet": "fd00:1:1::/64",
                    "gateway": "fd00:1:1::1"
               }
          ],
          "ipv6_enabled": true,
          "internal": false,
          "dns_enabled": true,
          "ipam_options": {
               "driver": "host-local"
          },
```

## docker-compose.yml

The bit you've been waiting for..

```
# Postgres config using a .env file, podman secrets and IPv6 network

services:
  database:
    image: 'richard/postgres:latest'
    container_name: 'postgres_test'
    ports:
      - 5432:5432
    env_file:
      - .env 
    userns_mode: keep-id:uid=999,gid=999
    networks:
      - postgres-network
    secrets: 
      - pgsqladmin
    environment:
      - POSTGRES_PASSWORD_FILE=/run/secrets/pgsqladmin
      - PGDATA=/var/lib/postgresql/data/testing_times
    volumes:
      - pgdata:/var/lib/postgresql/data
      - /data/dev/postgres_setup/init-user-db.sh:/docker-entrypoint-initdb.d/init-user-db.sh
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -d postgres"]
      interval: 10s
      timeout: 5s
      retries: 5 

volumes:
  pgdata:
    driver: local 
    driver_opts:
      type: none
      device: /data/dev/volumes/postgres_data
      o: bind
    #name: 'pgdata'

networks:
  postgres-network:
    driver: bridge
    #name: 'postgres-network'

secrets:
  pgsqladmin:
    external: true
```

The final few lines show the addition of secrets to the compose file. The following few lines of service definition is the key:
```
 secrets: 
      - pgsqladmin
    environment:
      - POSTGRES_PASSWORD_FILE=/run/secrets/pgsqladmin
 ```     
pgsqladmin is a literal substitution, and POSTGRES_PASSWORD_FILE is a postgres env variable. Note the following: POSTGRES_PASSWORD and POSTGRES_PASSWORD_FILE are mutually exclusive. So you can't specify POSTGRES_PASSWORD in the .env file at the same time.

If you so wish you can inspect where this ends up in the container:
```
-> podman exec -it postgres_test bash
postgres@80277c77241d:/$ ls -ltra /run/secrets
total 4
-r--r--r-- 1 root root  7 Feb  2 16:55 pgsqladmin
drwxr-xr-x 1 root root 60 Feb  2 16:55 ..
drwxr-xr-x 2 root root 60 Feb  2 16:55 .
```
and this contains the unencrypted plain-text password. 

The volume section specifies the location of the **init-user-db.sh** script but of note is the use of PGDATA - this creates a subvolume in the bind area, I don't think you'd ever really choose to run multiple database servers on the same storage platter, just created more DBs in your server if you need more stuff at the SOHO level, however, this does give a straight-forward route to copy the data and run newer versions of postgres to QA it before actually pulling the trigger, specifically the file system will at least be logical.

## The magic of userns_mode

Podman doesn't run containers as root, that's the provenance of dockerd. Postgres performs a number of interesting things like chown's that will cause you grief.

```
podman chown: changing ownership of '/var/lib/postgresql/data': Operation not permitted
```

Oh no! This took me a little while to figure out - postgres is a user and groud id of 999 within the container, but this doesn't exist on the host, which is to be expected. Even if you did try to fudge a user it will be below the default /etc/subuid and /etc/subgid range. This is the magic that makes it work:

```
    userns_mode: keep-id:uid=999,gid=999
```
This overrides the subuid and subgid range that exists and allows podman to map uid/gid with 999 to my host id.

```
richard@microserver:/data/dev/volumes/postgres_data> ls -tlra
total 4
drwxr-xr-x  3 richard users   27 Jan 29 15:29 ..
drwxr-xr-x  3 richard users   35 Jan 29 15:40 .
drwx------ 19 richard users 4096 Feb  2 16:55 testing_times
```

Excellent!

One final issue comment as you should have a clean install at this point:
```
richard@microserver:/data/dev/postgres_setup> podman logs postgres_test
chmod: changing permissions of '/var/run/postgresql': Operation not permitted
The files belonging to this database system will be owned by user "postgres".
```
This is a different error from the pgdata volume mapping. After scouring some forums, apparantly this is expected behaviour for running as non-root, as confirmed by a postgres SME, whom says they specifically don't fail on this chmod on the actual process. Ok well you could check if you are root before trying to do it couldn't you.. but anyway, they said it's good so I'm good :)


