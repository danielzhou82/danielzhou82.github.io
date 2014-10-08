---
layout: post
title: "Containerize Syslog-ng"
date: 2014-09-30 14:22:55 +0800
comments: true
categories: Linux
keywords: LXC, container, syslog
description: how to enable multiple syslog-ng instances for multiple LXC containers.
---

I have a pretty old LXC configuration, with `kernel 2.6.32` and `LXC 0.7`. The problem is I wasn't able to make all syslog-ng instances in both host and containers work. The syslog-ng daemon is running, however isn't writing anything to log files.

The culprit is the `/dev/log` UNIX socket.

The syslog in UNIX system is running in client/server mode:

 - client is the `syslog()` interface in glibc. `syslog()` connects to socket `/dev/log` and send messages via the socket.
 - and server is the listener of socket `/dev/log`, such as syslog-ng, rsyslog.

<!--more-->
This is how syslog-ng use `/dev/log`:

 - check if `/dev/log` is already there, if so unlink it;
 - create `/dev/log` with socket/bind.

And this is how I mount `/dev` in container config:

```
lxc.mount.entry=/dev /var/lxc/ct1/rootfs/dev none defaults,bind 0 0
```

Because `/dev` is bind-mounted, `/dev/log` is shared among all containers, each time when syslog-ng instance starts, the old `/dev/log` is unlinked. The removal apparently will break the current syslog-ng instance listening on it. So at last only one syslog-ng instance (the last one) can work without problem.

Finally I made it work.

First I changed syslog-ng configuration files for both host and containers:

```
# cat /etc/syslog-ng/syslog-ng.conf
...
source s_sys {
        file ("/proc/kmsg" program_override("kernel: "));
        unix-stream ("/var/run/syslog-ng.sock");
        internal();
};
...
```

It means syslog-ng will listen to an alternative socket at `/var/run/syslog-ng.sock` instead of `/dev/log`. Since `/var` is a container private directoy, each syslog-ng can listen without interfering with each other.

At the client end, glibc is still using `/dev/log` to write log message, so I made `/dev/log` a symbol link:

```
ln -sf /var/run/syslog-ng.sock /dev/log
```

Now when application calls `syslog()`, it connects and send messages to `/var/run/syslog-ng.sock`. Syslog-ng who is listening on the socket will handle the message correctly.

To make the change permanent, I changed the start routine in `/etc/init.d/syslog-ng`:

``` bash
start() {
        echo -n "Starting syslog-ng: "
        if [ ! -h /dev/log ]; then
                rm -f /dev/log 2>/dev/null
                ln -sf /var/run/syslog-ng.sock /dev/log
        fi
        ulimit -u 600000
        daemon $binary
        RETVAL=$? 
        echo
        [ $RETVAL -eq 0 ] && touch /var/lock/subsys/syslog-ng
}
```

It makes sure `/dev/log` is a correct symbol link when syslog-ng starts.
