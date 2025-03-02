---
draft: false
author: "Tamas Rebeli"
title: Isolating resource consumption for PostgreSQL instances in Linux – Part 1
description: Overview of core Linux techniques, and using systemd to set resource limits per PostgreSQL instance
publishDate: 2025-03-02
date: 2025-02-15
Image_webp: images/blog/isolating-postgres-instances-cover.webp
image: images/blog/isolating-postgres-instances-cover.png
tags:
- linux
- cgroups
- systemd
- resource control
- resource limits
- nice
- scheduling
- postgresql
---

In a traditional, on-premises enterprise environment, a single bare-metal server or virtual machine may host multiple PostgreSQL server instances. Even though they share the same machine, they often belong to different applications, projects or organizational units, and are required to be isolated in terms of utilizing system resources like RAM, CPU and disk I/O, usually for the purpose of business management.

PostgreSQL is an excellent database management system in general. On the other hand, its built-in resource management – if you can even call it that – mostly operates at a lower level than what would be ideal for setting up such an isolation in a straightforward manner. You can enforce resource consumption limits like memory utilization and degree of parallelism for certain SQL operations. Most of these settings, however, do not impose global hard limits for the PostgreSQL server instance: not for databases nor for users or even queries for that matter.

Moreover, setting up priorities between users, queries or databases – so that one may consume more resources than others – is a feature completely missing from vanilla PostgreSQL. Then again, even if prioriziation were possible at those levels, it is generally recommended to deploy a separate PostgreSQL server instance per database. The single-instance multi-database configuration is desired only if those databases logically belong together, and may be managed – stopped, started, backed up, upgraded etc. – as a single unit in terms of business functionality.

Does it all mean that high-level isolation is impossible? Thankfully, it does not.

> It is possible to isolate PostgreSQL instances, but as with quite a few other aspects in PostgreSQL, limiting overall resource consumption or setting up priorities needs to be implemented at the operating system level.


After a run-down of some basic resource control techniques in Linux, we are going to cover how the desired level of resource management may be achived via systemd, and how to configure it specifically for PostgreSQL.


# Core Linux technologies for resource management
Sharing server resources between users has always been the norm since the very early ages of computing. Controlling shared resource consumption came as a natural requirement. Some of the older tools discussed here date back to the 1970's, but even the most recent core technology originated as early as the late 2000's. That means there's a good chance you've already heard about the following technologies. Let's take a quick look.

## Classical resource limits
What first comes to mind is the age-old resource limiting solution commonly referred to as [ulimit](https://pubs.opengroup.org/onlinepubs/9699919799/functions/ulimit.html). POSIX originally used this name for process limits, but later deprecated the name in favour of a new one called *rlimit* (resource limit).

On the one hand, GNU/Linux, resource limiting library functions are also implemented by [glibc](https://www.gnu.org/software/libc/manual/html_node/Limits-on-Resources.html) with **`rlimit`** in their names, like the [`prlimit()`](https://man7.org/linux/man-pages/man2/prlimit.2.html) system call,
which can set resource limits per process. On the other hand, the shell built-in to get and set resource limits on the command line in an OS user session (for child processes forked by the shell) somehow kept the old name `ulimit` in Linux. In contrast with the shell built-in, the program that can change limits for already running processes is again called [`prlimit`](https://man7.org/linux/man-pages/man1/prlimit.1.html), following the new naming convention. The consistency of naming limit related functions and programs is a mess.

Linux also has something called *security limits*, which is basically a [PAM](https://www.man7.org/linux/man-pages/man3/pam.3.html) wrapper around *rlimit* calls to automate `ulimit` for OS user session creation, traditionally configured in [`/etc/security/limits.conf`](https://www.man7.org/linux/man-pages/man5/limits.conf.5.html).

With classical resource limits, you can set limits for individual processes per OS user session. They are not per-user settings, they are per-process settings. Linux does not aggregate resource consumption of processes by owner user at all. Child processes inherit the limits set for the parent. Limits set by the shell built-in `ulimit` apply to various resources. One may enforce the maximum CPU time in seconds the process can use (`-t` or `cpu`), the maximum virtual memory size (address space) (`-v` or `as`), and the maximum stack size (`-s` or `stack`) the process can allocate. Please note that limiting physical memory usage (resident set size) (`-m` or `rss`) was disabled a long time ago, in kernel 2.4.30. Setting that limit will have no effect.

You might face the need to change some of these settings every now and then. For example, you might want to increase the stack size to allow a higher value for the `max_stack_depth` PostgreSQL server parameter. Classical resource limit settings, however, are not instrumental to isolating PostgreSQL instances. Setting a limit for the CPU time in seconds is not a great idea for a relational database in general. It's also ill-advised to limit virtual memory size of a process instead of limiting actual RAM usage, but the actual RAM usage called RSS size cannot be limited in Linux. There is no way to limit I/O usage either.


## Nice value
As for setting priorities, another long-standing POSIX concept called [*nice value*](https://pubs.opengroup.org/onlinepubs/9799919799/functions/nice.html) may come to mind.

A "nicer" process consumes less resources, which means increasing niceness will make the process lower priority, or in other words the CPU scheduler will favour the process less. Child processes will inherit their priority and nice values from their parents.

In GNU/Linux, you can use the [`nice`](https://man7.org/linux/man-pages/man1/nice.1.html) command to run a program with a specified nice value, and the [`renice`](https://man7.org/linux/man-pages/man1/renice.1.html) command to interactively change the nice value for a given process. The same result can be achieved programmatically with the [`setpriority()`](https://man7.org/linux/man-pages/man2/setpriority.2.html) system call.

Only root can increase the priority (set a negative nice value) for a process, but anyone can decrease the priority (set a positive nice value) of their own process - up to the configured limit (see below). The new priority is calculated by the formula: *current priority + nice value*. 

Unfortunately, the terms "nice value" and "priority" are sometimes used interchangeably, which can get quite confusing. For example, the `renice` command refers to the nice value as "priority", and in classical resource limits, the two terms are even more conflated.

Classical resource limits also include limits for both nice value and priority, to be applied at the user session level. With `ulimit`, you can set a limit called "scheduling priority" (`-e`), which limits the maximum priority of any processes started in scope of the affected session. In `limits.conf`, however, you can define two different limits: `priority` and `nice`. If you set a hard limit for `priority`, it is interpreted and set directly as the nice value, which in turn affects scheduling priority as usual. If you set a hard limit for `nice`, it changes the scheduling priority limit (`-e`) that ist the maximum priority users can set for their processes.

By adjusting the nice value of a PostgreSQL client backend process, you can prioritize the given database session over others. You can also adjust the nice value for the `postgres` process, the parent of all other processes in the PostgreSQL instance. The new nice value prioritizes that particular instance over others.

Prioritizing disk I/O for processes is also possible in Linux via [`ionice`](https://man7.org/linux/man-pages/man1/ionice.1.html) on the command line, and via the [`ioprio_set()`](https://man7.org/linux/man-pages/man2/ioprio_set.2.html) system call programmatically, but the technique requires the kernel to use the [CFQ scheduler]( https://www.kernel.org/doc/Documentation/block/cfq-iosched.txt) for the disk devices used by PostgreSQL. Although using CFQ scheduler is usually feasible, it may not be the default I/O scheduler in your distribution of choice, and might just not be the best option for your particular workload.


## Control groups
A more recent and more featureful resource management solution implemented in the Linux kernel is *cgroups* ([Control Groups](https://docs.kernel.org/admin-guide/cgroup-v1/cgroups.html)), originally developed by Google engineers, which ran by the name *Process Containers* at that time. It is one of the main pillars of today's container technologies running on Linux, along with Linux namespaces.

Cgroups allows processes to be organized into a hierarchy of groups, and provides control over resource consumption for those groups of processes. 
Each process inside a cgroup has the limits of that cgroup imposed. 

A wide variety of controllers are available in cgroups to limit the consumption of different resource types, including CPU, memory and block I/O. These controllers provide significant improvements for fine-grained resource consumption control over earlier solutions.

In 2015, a new version of cgroups, [*cgroups-v2*](https://docs.kernel.org/admin-guide/cgroup-v2.html) made its way into the Linux kernel (in version 4.5). Cgroups v2 is also commonly referred to as the *unified hierarchy*, because this feature was probably the most significant change overall, aiming simplicity over unnecessary flexibility. While the more flexible v1 earlier assigned each controller its own mount point under `/sys/fs/cgroup/`, *unified hierarchy* mounts all groups under the single `/sys/fs/cgroup` hierarchy. On top of that, other changes were introduced as well affecting I/O and memory control. Newer Linux distributions are using cgroups-v2 now, but from time to time you may still meet Linux installations with cgroups v1 still.


# Using systemd for resource management
You can use those core Linux technologies introduced above in a more centralized (and perhaps less confusing) manner by an even more recent technology, [*systemd*](https://man7.org/linux/man-pages/man1/init.1.html). It is an init system and service manager for GNU/Linux, originally developed by Red Hat, and later adopted by all major Linux distributions, following long and heated debates. Systemd is now wide-spread, and you must get familiar with it if you haven't done so.

Part 2 focuses on resource management solutions based on systemd. But just before we get to that, let's briefly look at how systemd organizes its managed processes.


## Systemd unit types
Systemd manages different types of units. Important types for the purpose of this article are: *service*, *scope* and *slice*.

### Services and scopes
You are most probably familiar with service units. A *service* runs one or more processes, or in other words, a service is a logical group of processes. A service unit is defined in its [*unit file*](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html) named `<something>.service`.

A *scope* is created automatically by systemd for each user session (and some other things) running in Linux. A scope is also a logical group of processes. The difference between service and scope is that the processes in a scope are not started directly by systemd, they are merely assigned to the scope by systemd. That is also why you cannot define a scope in a unit file.

You can use the `systemctl --type scope` command to list scopes.

    root@sysagnostic:~# systemctl --type scope
    UNIT                 LOAD   ACTIVE SUB     DESCRIPTION
    init.scope           loaded active running System and Service Manager
    session-357691.scope loaded active running Session 357691 of user root


### Slices
A *slice* is a group of services and scopes, not of individual processes directly: it is one level higher. A slice is defined in its own unit file `<name>.slice`.

You can use the `systemctl --type slice` command to list slices.


    root@sysagnostic:~# systemctl --type slice
    UNIT                         LOAD   ACTIVE SUB    DESCRIPTION
    -.slice                      loaded active active Root Slice
    system-getty.slice           loaded active active system-getty.slice
    system-modprobe.slice        loaded active active system-modprobe.slice
    system-systemd\x2dfsck.slice loaded active active system-systemd\x2dfsck.slice
    system.slice                 loaded active active System Slice
    user-0.slice                 loaded active active User Slice of UID 0
    user.slice                   loaded active active User and Session Slice


You may have noticed that slices are organized into a hierarchy, which means slices may contain other slices. 

The root slice is `-.slice`. 

Below the root slice, there are at least two other slices: 
- the `system.slice` (holding all system services) and
- the `user.slice` (holding all user session scopes).

The name of a slice indicates the path to it in the hierarchy. In general, slice names look like this: `<parent>-<child>-<grandchild>.slice`. 


## Cgroups managed by systemd
Now let's take a look at how cgroups is integrated into systemd.

Systemd maintains a default cgroups hierarchy, following its own hierarchy of slices, services and scopes.

You can use the `systemd-cgls` command to list cgroups created by systemd.

    root@sysagnostic:~# systemd-cgls --no-pager
    Control group /:
    -.slice
    ├─user.slice 
    │ └─user-0.slice 
    │   ├─session-357691.scope 
    │   │ ├─287844 sshd: root@pts/0
    │   │ ├─287875 -bash
    │   │ └─287972 systemd-cgls --no-pager
    ├─init.scope 
    │ └─1 /sbin/init
    └─system.slice 
      ├─systemd-udevd.service 
      │ └─263 /lib/systemd/systemd-udevd
      ├─cron.service 
      │ └─475 /usr/sbin/cron -f
      ├─systemd-journald.service 
      │ └─240 /lib/systemd/systemd-journald
      ├─ssh.service 
      │ └─543 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
      ├─system-getty.slice 
      │ └─getty@tty1.service 
      │   └─525 /sbin/agetty -o -p -- \u --noclear tty1 linux
      …

You can use the `systemctl status` command for a unit to see where it is located in the cgroups hierarchy.

As you might have guessed, the cgroups hierarchy maintained by systemd allows you to define cgroups resource limits for slices, services and scopes. Such definitions go into the corresponding unit files (`<name>.slice` or `<name>.service`), or limits can be set interactively for an already runing unit in a dynamic fashion (that is the only option for scopes).

Available parameters related to CPU, RAM and I/O resource control are all explained in the [Resource control unit settings](https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html) page of the systemd documentation.

You can use the `systemctl show` command to see the current settings for a unit, including the ones related to resource control.

    systemctl show <name>.service | grep -E "^CPU|^Memory|^IO"

We will come back to some specific resource control settings shortly. Before we do that, let's touch upon the older limiting techniques once more, to see how they integrate into systemd.


## Classical resource limits managed from systemd
A systemd service unit can have rlimit (ulimit) parameters defined for it, which apply to all of the processes included in that unit. 
There is a [mapping between ulimit and systemd parameter names](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#id-1.12.2.1.17.6).

To check rlimit settings for a systemd unit, you can use the `systemctl show` command.

    systemctl show <name>.service | grep "^Limit"

(To check rlimit settings for an individual process, you can read the `/proc/<PID>/limits` file.)

## Nice configuration managed by systemd
Nice value can also be defined via systemd service unit files.

To check the nice value setting for a systemd unit, you can use the `systemctl show` command.

    systemctl show <name>.service --property=Nice

To check the priority and nice value of an individual process, you can use the `ps -o pid,comm,pri,ni -q <PID>` command.

# To be continued...
In [Part 2](/blog/isolating-postgres-instances-p2) of this article, we will look at how to configure isolation of resource consumption for PostgreSQL via systemd.
