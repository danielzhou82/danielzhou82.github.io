---
layout: post
title: "On the Confusing Ulimit"
date: 2014-09-30 13:01:14 +0800
comments: true
categories: 
---
#On the Confusing ulimit

##1. ulimit and rlimit

Literally, ulimit is per user resource limit, such like the total process number allowed to spawn, and the total number of file allowed to open etc. In Linux, user delegates process to apply system resource. To understand ulimit, one must get clear idea of rlimit of process.

The definition of rlimit:

```
struct rlimit {
    rlim_t rlim_cur;  /* Soft limit */
    rlim_t rlim_max;  /* Hard limit (ceiling for rlim_cur) */
};
```

rlimit structure is used to describe resource limit of process (not accurate actually, will explain later). rlim_cur is current active limit and rlim_max is the ceiling value of rlim_cur.

There are various types of resource in Linux OS, hence an array rlim[RLIM_NLIMITS] is included (indirectly) in task_struct (the process descriptor). Each different type of resource limit is referenced by the array index. For example, rlim[RLIMIT_NPROC] is the maximum number of process and rlim[RLIMIT_NOFILE] is the maximum number of files.

```
struct task_struct ->
    struct signal_struct ->
        struct rlimit rlim[RLIM_NLIMITS]
```

Linux kernel provides setrlimit()/getrlimit() syscalls for process to write/read its rlimit of specific resource.

Now we know the rlimit is attached to process, then where are the resource counters? In Linux, they are in user_struct (the user descriptor).

```
struct task_struct ->
    struct cred *real_cred ->
        struct user_struct *user -> 
            process counter: atomic_t processes
            file counter: atomic_t files
```

So how the limitation works? Take RLIMIT_NPROC as an example, in function fork() in kernel, we have below snippet:

```
        retval = -EAGAIN;
        if (atomic_read(&p->real_cred->user->processes) >=
                        p->signal->rlim[RLIMIT_NPROC].rlim_cur) {
                if (!capable(CAP_SYS_ADMIN) && !capable(CAP_SYS_RESOURCE) &&
                    p->real_cred->user != INIT_USER)
                        goto bad_fork_free;
        }
```

The kernel compares the counter in user_struct with the rlimit in task_struct, then reject the fork request by returning EAGAIN if rlimit is exceeded, otherwise gives green light and bump up the counter. As special cases, the check is bypassed if the process has capabilities of CAP_SYS_ADMIN or CAP_SYS_RESOURCE or the owner of the process is root.

```
do_fork()
    copy_process()
        copy_creds() // inherit parent's credential
        check processes and rlimit // see above
        processes ++
```

We can see from above call stack that child process will inherit the credential (user_struct as well) of parent when doing fork(), which indirectly inherits all rlimit settings as well.

The rlimit is per process, while the resource counters are per user. I guess this is the root of all confusion about ulimit and rlimit. Say we have user U and his two processes P1 and P2, the RLIMIT_NPROC are 100 for P1 and 200 for P2. Then we have:

- when counter C < 100, both P1 and P2 are allowed to spawn new processes;
- when 100 <= C < 200, only P2 is allowed to spawn new processes;
- when C > 200, both P1 and P2 are not allowed to spawn new processes.

From processes' perspective, rlimit is not the quantity of resource a process can own. To check how much resource one processes can use, we must know how much other processes from the same user have alreay taken. This is one important reason why some application like Apache and Mysql run under their own user.

##2. how rlimit inherited?

It's also confusing when it comes to the rlimit inheritance problem. Given a process, where are all the rlimit values from? And how are we supposed to change the values?

- init process: kernel decides most of rlimit values;
- System Daemons: inherits from init, however may change rlimit by calling setrlimit()
- Login Shell: use the values defined in /etc/security/limits.conf if exist, inherits from init for the rest;
- Process spawned in Shell: inherits from Shell

### 2.1 process 1: init

Process 1 (init) is very special. Kernel finishes most of environment setups, and loads /sbin/init from user land. Running as a normal Linux process, /sbin/init is allowed to call setrlimit() to change the rlimit value from kernel. For example, SysVinit code sets RLIMIT_CORE to 0 at early stage.

We can read rlimit values of process init from procfs:

```
\# cat /proc/1/limits
...
Max processes             193115               193115               processes 
Max open files            1024                 4096                 files     
...
```

So why the RLIMIT_NOFILE of init is 1024? Actually it's hardcoded by kernel.

For your reference, see INIT_RLIMITS macro in kernel source, which initializes rlim array for process init:

```
\#define INR_OPEN_CUR 1024
\#define INR_OPEN_MAX 4096
\#define INIT_RLIMITS                                                    \
{                                                                       \
...
        [RLIMIT_NPROC]          = {              0,              0 },   \
        [RLIMIT_NOFILE]         = {   INR_OPEN_CUR,   INR_OPEN_MAX },   \
...
}
```

For RLIMIT_NPROC, kernel fuction fork_init() decides its value by half of global variable max_threads, and max_threads correlates to the capacity of physical memory. So we can say RLIMIT_NPROC of init depends on hardware configuration.

```
init_task.signal->rlim[RLIMIT_NPROC].rlim_cur = max_threads/2;
init_task.signal->rlim[RLIMIT_NPROC].rlim_max = max_threads/2;
init_task.signal->rlim[RLIMIT_SIGPENDING] = init_task.signal->rlim[RLIMIT_NPROC];
```

You may notice that we may have different RLIMIT_NOFILE for RHEL5 and RHEL6. This is due to below kernel commit:

>commit 0ac1ee0bfec2a4ad118f907ce586d0dfd8db7641
>Author: Tim Gardner <tim.gardner@canonical.com>
>Date:   Tue May 24 17:13:05 2011 -0700

>    ulimit: raise default hard ulimit on number of files to 4096

###2.2 system daemons

System daemons are brought up by init according to predefined configurations, in a non-interactive way. Hence they normally inherits rlimit settings from init directly. However system daemons may call setrlimit() to change rlimit values, depending on the requirement. Take syslog-ng as an example, we can see setrlimit() invocation in C source code:

```
limit.rlimit_cur = 4096;
setrlimit(RLIMIT_NOFILE, &limit);
```

Then the rlimit of syslog-ng looks like:

```
\# cat /proc/3268/limits 
Limit                     Soft Limit           Hard Limit           Units     
...
Max processes             193115               193115               processes 
Max open files            4096                 4096                 files     
...
```

The soft limit of RLIMIT_NOFILE is now 4096, other than init's 1024.

###2.3 Login Shell

In normal interactive shell spawn procedure, a process will be forked and /bin/login will be loaded for user authentication, and then PAM plugins will set up rlimit by reading values from /etc/security/limits.conf. All child processes of this shell will inherit these rlimit settings.

/etc/security/limits.conf is the main configuration file for system administrator to set up default rlimit, however you need to know this method doesn't apply to all scenarios.

In man page of limits.conf:

>       Also, please note that all limit settings are set per login. They
>       are not global, nor are they permanent; existing only for the
>       duration of the session.

Internally, /etc/security/limits.conf is actually the configuration file of one PAM plugin pam_limits.so, and has nothing to do with Linux Shell. As part of login procedure, PAM invokes system-auth submodule, which in turn involves pam_limits.so plugin, then it's plugin's responsibility to get rlimit value from limits.conf.

```
\# cat /etc/pam.d/login 
...
session    include      system-auth
...
\# cat /etc/pam.d/system-auth
...
session required pam_limits.so
...
```

Please note PAM is user oriented, so it has no effect on system daemons, i.e., the rlimits of processes that are already running stay intact after limits.conf changed.

##3. how to configure rlimit


In order to configure rlimit, Linux OS provides system API setrlimit(), Shell provides ulimit command line, and PAM provides limits.conf interface.

For non-root users, it's not allowed to change the hard limit or increase the soft limit, i.e., the only thing non-privileged user can do is lowering down the soft limit.

Please note no matter how big the values provided by user is, kernel will make sure it won't exceed some global upper bound. For example, RLIMIT_NOFILE is not allowed to exceed the value defined in /proc/sys/fs/nr_open, and RLIMIT_NPROC is not allowed to exceed the value defined in /proc/sys/kernel/threads-max.

In next sections we will talk about how to set up rlimit without calling setrlimit() API.

###3.1 Shell

Let's go back to ulimit. First of all, ulimit is an internal command of Shell and runs within Shell's address space. Internally, ulimit is just a wrapper of setrlimit()/getrlimit() API, so we can say ulimit is nothing but rlimit configurator of Shell.

From the output of stace tool, ulimit -a actually calls getrlimit() to print out all rlimit values of Shell. Note we have to create a new shell instance to support strace because ulimit is internal command of Shell.

```
\# strace bash -c "ulimit -a"
...
getrlimit(RLIMIT_NOFILE, {rlim_cur=65535, rlim_max=65535}) = 0
write(1, "open files            "..., 43open files                 (-n) 65535
) = 43
...
getrlimit(RLIMIT_NPROC, {rlim_cur=20*1024, rlim_max=20*1024}) = 0
write(1, "max user processes      "..., 43max user processes         (-u) 20480
) = 43
...
```

How to check other users' ulimit settings? We have a trick here: create a shell instance under other user's name and run ulimit in the new shell.

```
$ su admin -c "ulimit -a"
```

To set rlimit values, we run "ulimit -S". For example:

To enable core dump with unlimited dump size:

```
$ ulimit -Sc unlimited
```

and disable core dump:

```
$ ulimit -Sc 0
```

Obviously ulimit -S doesn't affect running processes.

###3.2 Login Shell

For login shell, besides running ulimit explicitly, we can realize automatic setup by changing the rlimit values in /etc/security/limits.conf.

###3.3 system daemons

If a system daemon doesn't change rlimit in its implementation (C code probably), we can add ulimit to the control script of the daemon (like /etc/init.d/syslog-ng for syslog-ng). Because the daemon itself is spawned from the control script, so it surely will inherit rlimit setting of the shell instance. Please also note this shell instance is not a login shell, hence PAM will stay out.

>Hint: upstart provides stanzas to configure rlimit for upstart services (not apply to legacy SysVinit daemons), see http://upstart.ubuntu.com/wiki/Stanzas#limit.
 
###3.4 processes that are already running

For the processes in running state, how to change rlimit setting dynamically?

The answer is probably the limits file beneath /proc/<pid>/. This file is read/write and root user (or anyone with sudo privilege) can write formatted string to it to change rlimit value.

This is how root user adjusts RLIMIT_NPROC of process (pid):

```
\# echo -n \"Max processes=131072:131072\" >/proc//limits
```

and if user is non-root with sudo privilege:

```
$ sudo su -c "echo -n \"Max processes=131072:131072\" >/proc//limits"
```

I hope this article could dispel some of the confusion you may have and please bear with my poor English.

