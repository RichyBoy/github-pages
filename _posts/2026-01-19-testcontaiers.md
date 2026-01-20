---
title: "Rootless podman, testcontainers, python and WSL"
date: 2026-01-19
layout: post
tags: podman rootless testcontainers python WSL
---

# Testcontainers and rootless podman

A short post.. in principle spinning up and pulling down containers has significant appeal for testing certain data sets I have.. the so-called "ground-hog" data tests as we used to call them. As I can't abide running as root I naturally want to run rootless podman. In other posts I have explained how various tools can be configured but there's still a couple of gotchas with WSL.

It's interesting having a look around the web, there's a lot of people posting all kinds of stuff on the subject, even installing qemu to created podman machines - all quite wild, and not needed. Well you can do but I personally don't see the need to "remotely" connect from a podman desktop install to a wsl install.. what are you doing!

First up, podman APIs need to communicate through the ```podman.sock``` Unix socket, but more commonly the podman service is activated on-demand when something connects to the socket. 

In rootless mode (that is, user mode) the user will need to create ```podman.sock``` and it needs to live in ```/run/user/$(id -u)/podman/podman.sock```. However to enable the whole service podman (for your user) needs to be configured and your user needs a copy of a couple of systemd files.

```
cp  /usr/lib/systemd/user/podman.socket ~/.config/systemd/user/podman.socket
cp  /usr/lib/systemd/user/podman.sservice ~/.config/systemd/user/podman.service

systemctl --user daemon-reload
systemctl --user enable --now podman.socket
```
The first command will reload the current user's systemd configs, and the second command will enable the socket. It should be available here:
```/run/user/$(id -u)/podman/podman.sock``` and your uid is probably 1000. Other sources sometimes quote $(XDG_RUNTIME_DIR) and XDG variables are setup during your login (via pam).

If it hasn't appeared I can only suggest shutting down wsl from the command prompt with ```wsl --shutdown```.

We're half-way home. If you've read one of my other articles you will be aware there is an issue with nft firewall tables which is what podman tries to use by default; in my other article I'd overridden them for local containers but as we're spinning them up via socket connection it actually defaults to the system config (annoyingly). Although maybe my user config is broken. The below works anyway, and this addresses errors such as ```netavark (exit code 1)```

If it doesn't exist create the following file:
```
 /etc/containers/containers.conf
 ```
 and add this to it:
 ```
[network]
firewall_driver="iptables"
 ```

This time you'll certainly have to bounce WSL and in fact I rebooted as it broke VS code <-> WSL sockets entirely.

##  One more task for testcontainers

You will need to let testcontainers know about the "docker host", which is podman and not docker. I found this unreliable:

```
cat ~/.testcontainers.properties

docker.host=unix://${XDG_RUNTIME_DIR}/podman/podman.sock
```

Eventually I resorted to this in my ```.bashrc```
```
export TESTCONTAINERS_RYUK_DISABLED=true
export DOCKER_HOST=unix:///run/user/1000/podman/podman.sock
```
Yes yes, I know I hard-coded it!

## A small note on testcontainer
The documentation is crap and the examples don't work. This is in reference to [https://testcontainers.com/guides/getting-started-with-testcontainers-for-python/](https://testcontainers.com/guides/getting-started-with-testcontainers-for-python/)

They will run though.. once you've fixed them up. Recall for each python module you will need a empty ```__init__.py``` file so add one to customers directory.

In customers/customers.py change the following:
```
def get_all_customers() -> list[Customers]:
    with get_connection() as conn:
        with conn.cursor() as cur:
            cur.execute("SELECT * FROM customers")
            return [Customer(cid, name, email) for cid, name, email in cur]


def get_customer_by_email(email) -> Customers:
    with get_connection() as conn:
        with conn.cursor() as cur:
            cur.execute("SELECT id, name, email FROM customers WHERE email = %s", (email,))
            (cid, name, email) = cur.fetchone()
            return Customers(cid, name, email)
```
The code has typo'd the class name which is Customers and not Customer.

Thirdly, there's a missing pip depdendency.

```
pip install "psycopg[binary,pool]"
```
That will fix missing pyscopg binary dependency.

It will probably run now:

```
platform linux -- Python 3.13.9, pytest-9.0.2, pluggy-1.6.0
rootdir: /mnt/dev/testc
testpaths: ./tests/
collected 0 items
```
But it's still broken, collected 0 items. You can force it to actually run and collect the two tests by commenting out the annotations in the tests/test_customers.py fie. 
Now it will run, but it will fail and I really don't know how people publish trash like this.

If you want to actually see testcontainers running at the basic level, try this link here: [https://pytest-with-eric.com/introduction/pytest-collected-0-items/](https://pytest-with-eric.com/introduction/pytest-collected-0-items/)

The next step is to get it running with a container, but a different post for a different day. The main thing: everything should work now with podman and it's just down to learning/using testcontainers. Probably best to start with nginx over a database methinks!


