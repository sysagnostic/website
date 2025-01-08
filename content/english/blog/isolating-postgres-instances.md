---
draft: true
author: "Tamas Rebeli-Szabo"
title: Isolating resource consumption for PostgreSQL instances in Linux
description: Overview of core Linux techniques, and using systemd to set resource limits per PostgreSQL instance
publishDate: 2025-01-15
date: 2024-01-08
tags:
  - Linux
  - cgroups
  - systemd
  - resource control
  - nice
---

# Isolating resource consumption for PostgreSQL instances in Linux
In a traditional, on-premises enterprise environment, a single bare-metal server or virtual machine may host multiple PostgreSQL server instances. Even though they share the same machine, they often belong to different applications, projects or organizational units, and are required to be isolated in terms of utilizing system resources like RAM, CPU and disk I/O, usually for the purpose of business management.

PostgreSQL is an excellent database management system, but its built-in resource management – if you can even call it that – is mostly at a lower level than what would be ideal for setting up such an isolation in a straighforward manner. You can set some resource consumption related parameters to control things like memory utilization and degree of parallelism for certain SQL operations, but most of those settings do not impose overall hard limits for the PostgreSQL server instance (nor for databases, users or even queries for that matter). 

Setting up priorities between users, queries or database so that one may consume more resources than others is a feature completely missing from community PostgreSQL too. Even if prioritiziation were possible at those levels, it is generally recommended to deploy a separate server instance per database, unless a set of databases logically belong together, and may be managed (stopped and started, backed up, upgraded etc.) as a unit in terms of business functionality.

Does it all mean such isolation is impossible? No, it does not, but as with quite a few other aspects in PostgreSQL database management, limiting overall resource consumption or setting up priorities needs to be implemented at the operating system level.

After a run-down of some basic resource control techniques in Linux, we are going to cover how resource management is made possible from systemd, and how to do it specifically for PostgreSQL.


## Core Linux technologies for resource management
Sharing resources between users on a server has always been the norm from the very early ages of computing, and controlling resource consumption came as a natural requirement. Some of the older tools discussed here date back to the 1970's, but even the most recent core technology originated in the late 2000's. That means you've probably heard about all of these. Let's take a quick look.

### Classical resource limits
What first comes to mind is the age-old resource limiting solution commonly referred to as [ulimit](https://pubs.opengroup.org/onlinepubs/9699919799/functions/ulimit.html). POSIX originally used this name for process limits, but later depricated it in favour of the new name *rlimit* (resource limit), which also refers to a more developed resource managment model.

In GNU/Linux, corresponding system calls are also implemented by [functions](https://www.gnu.org/software/libc/manual/html_node/Limits-on-Resources.html) with **`rlimit`** in their names, like [`prlimit()`](https://man7.org/linux/man-pages/man2/prlimit.2.html) system call,
which can set resource limits per process. Note that the shell built-in to get and set resource limits on the command line in an OS user session (for child processes forked by the shell) somehow kept the old name `ulimit` in Linux, but the command to change limits for already running processes is [`prlimit`](https://man7.org/linux/man-pages/man1/prlimit.1.html), following the new naming.

Linux also has something called *security limits*, which is basically a [PAM](https://www.man7.org/linux/man-pages/man3/pam.3.html) wrapper around *rlimit* calls to automate `ulimit` for OS user session creation, traditionally configured in [`/etc/security/limits.conf`](https://www.man7.org/linux/man-pages/man5/limits.conf.5.html).

With classic resource limits, you can set limits on individual processes in a given OS user session, but they are not per-user settings, they are per-process settings. Limits apply to resources like the maximum CPU time in seconds the process can use (`-t` or `cpu`), the maximum virtual memory size (address space) (`-v` or `as`) and the maximum stack size (`-s` or `stack`) the process can allocate, and the maximum number of files the process can open (`-n` or `nofile`). Note that limiting physical memory usage (resident set size) (`-m` or `rss`) was disabled a long time ago, in kernel 2.4.30. 

You might face the need to change some of these settings every now and then (eg. increase the stack size to allow a higher value for the `max_stack_depth` PostgreSQL server parameter). Classic resource limit settings, however, are not instrumental to isolating PostgreSQL instances. That is because a PostgreSQL server instance consists of a number of running processes, so limiting CPU usage per process is not a great idea (especially not CPU time), nor limiting virtual memory for individual processes is a viable method to impose an overall memory limit.


### Nice value
As for setting priorities, another long-standing POSIX concept called [*nice value*](https://pubs.opengroup.org/onlinepubs/9799919799/functions/nice.html) may come to mind.

A "nicer" process consumes less resources, which means increasing niceness will make the process lower priority, or in other words the CPU scheduler will favour the process less. Child processes will inherit their priority and nice values from their parents.

In GNU/Linux, you can use the [`nice`](https://man7.org/linux/man-pages/man1/nice.1.html) command to run a program with a specified nice value, and the [`renice`](https://man7.org/linux/man-pages/man1/renice.1.html) command to interactively change the nice value for a given process, while the same can be achieved programatically with the [`setpriority()`](https://man7.org/linux/man-pages/man2/setpriority.2.html) system call. 

Note that you can only increase the priority (set a negative nice value) as root, but a user can decrease the priority (set a positive nice value)for its own processes. The new priority will be caclulated as the current prority + nince value. 

Unfortunately, the terms "nice value" and "priority" are sometimes used interchangably, which can get quite confusing. The `renice` command refers to the nice value as "priority", and classical resource limits, the terminology is even more conflated.

Classical resource limits also include limits for both nice value and priority, to be applied at the user session level. In `ulimit`, there is a limit called scheduling priority (`-e`), which indeed limits the maximum priority. So far so good. In `limits.conf`, however, there are two limits: `priority` and `nice`. When a hard limit for `priority` is set, it is interpreted and set as the nice value for processes directly, which in turn affects priority as usual. When a hard limit for `nice` is set, it will manifest itself in the priority limit (`-e`), and will be the maximum priority you can set on the command line.

You can use the `ps` command to check nice value (`NI`) and priority (`PRI`) for processes (`ps -e -o pid,ni,pri,comm,args`).

Adjusting the nice value for a client backend process of a PostgreSQL server instance is a feasible method to give priority to a given database session. You can also adjust the nice value of the `postgres` process (which is the parent of all other processes in the PostgreSQL instance) to prioritize that particular instance over others. 

Prioritizing disk I/O for processes is also possible in Linux via [`ionice`](https://man7.org/linux/man-pages/man1/ionice.1.html) on the command line, and via the [`ioprio_set()`](https://man7.org/linux/man-pages/man2/ioprio_set.2.html) system call programmatically, but it requires the [CFQ scheduler]( https://www.kernel.org/doc/Documentation/block/cfq-iosched.txt) to be set for disk devices used by PostgreSQL, which may not be the default I/O scheduler in your distribution of choice, and might just not be the best option for your particular workload.


### Control groups
A more recent and more featureful resource management solution implemented in the Linux kernel is *cgroups* ([Control Groups](https://docs.kernel.org/admin-guide/cgroup-v1/cgroups.html)), originally developed by Google engineers, which ran by the name *Process Containers* at that time (and indeed it is one of the main pillars of today's container technology in Linux, along with namespaces.)

Cgroups allows processes to be organized into a hierarchy of groups, and allows control over resource consumption for those groups of processes. 
Every process inside a cgroup will have the limits of that cgroup imposed on it. 

Different controllers are available in cgroups to limit the consumption of different types of resources (CPU, memory, block I/O etc.).

In 2015, a new version of cgroups, [*cgroups-v2*](https://docs.kernel.org/admin-guide/cgroup-v2.html) made its way into the Linux kernel (version 4.5). It is also commonly referred to as the *unified hierarchy*, because that was a major change from v1 (in v1, every controller had a different mount point under `/sys/fs/cgroup/`), but other changes were introduced as well, including changes that affect memory control. Newer Linux distributions are using cgroups-v2 now, but you may very well see Linux installations with the original cgroups still.


## Using systemd for resource management
You can use those core Linux technologies introduced above in a more centralized (and perhaps less confusing) manner by an even more recent technology, [*systemd*](https://man7.org/linux/man-pages/man1/init.1.html). It is an init system and service manager for GNU/Linux, originally developed by Red Hat, and later adopted by all major Linux distributions, following long and heated debates. Systemd is now wide-spread, and you must get familiar with it if you haven't done so.

This article will focus on resource management solutions based on systemd. But just before we get to that, let's briefly look at how systemd organizes processes that it is managing.


### Systemd unit types
Systemd manages different types of units. Important types for the purpose of this article are: *service*, *scope* and *slice*.

#### Services and scopes
You are most probably familiar with service units. A *service* runs one or more processes, or in other words, a service is a logical group of processes. A service unit is defined in its [*unit file*](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html) `<name>.service`.

A *scope* is created automatically by systemd for each user session (and some other things) running in Linux. A scope is also a logical group of processes. (The difference is that the processes in a scope are not started directly by systemd, they are merely assigned to the scope by systemd. That is also why you cannot define a scope in a unit file.)

You can use the `systemctl --type` scope command to list scopes.

    root@sysagnostic:~# systemctl --type scope
      UNIT                 LOAD   ACTIVE SUB     DESCRIPTION
      init.scope           loaded active running System and Service Manag andr
      session-357691.scope loaded active running Session 357691 of user root


#### Slices
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


You may have noticed that slices are themselves organized into a hierarchy, which means there can be slices inside slices. 

The root slice is `-.slice`. 

Below the root slice, there are at least two other slices: 
 - the `system.slice` (holding all system services) and
 - the `user.slice` (holding all user session scopes).

The name of a slice indicates the path to it in the hierarchy. In general, slice names look like this: `<parent>-<child>-<grandchild>.slice`. 


### Cgroups managed from system
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
      ├─dbus.service 
      │ └─476 /usr/bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation --syslog-only
      ├─system-getty.slice 
      │ └─getty@tty1.service 
      │   └─525 /sbin/agetty -o -p -- \u --noclear tty1 linux
      └─systemd-logind.service 
        └─482 /lib/systemd/systemd-logind

(You can use the `systemctl status` command for a unit to see where it is located in the cgroups hierarchy.)

As you might have guessed, the cgroups hiearchy maintained by systemd allows you to define cgroups resource limits for slices, services and scopes. Such definitions go into the corresponding unit files (`<name>.slice` or `<name>.service`), or limits can be set interactively for an already runing unit in a dynamic fashion (for scopes this is the only option).

Available parameters related to CPU, RAM and I/O resource control are all explained in the [Resource control unit settings](https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html) of the systemd documentation. 

You can use the `systemctl show` command to see the current settings for a unit, including the ones related to resource control.

    systemctl show <name>.service | grep -E "^CPU|^Memory|^IO"

We will come back to some specific resource control settings shortly. Before we do that, let's touch upon the older limiting techniques once more, to see how they integrate into systemd.


### Classical resource limits managed from systemd
A systemd unit can have rlimit (ulimit) parameters defined for it, which apply to all of the processes included in that unit. 
There is a [mapping between ulimit and systemd parameter names](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#id-1.12.2.1.17.6).

To check rlimit settings for a systemd unit, you can use the `systemctl show` command.

    systemctl show <name>.service | grep "^Limit"

(To check rlimit settings for an individual process, you can read the `/proc/<PID>/limits` file.)

### Nice value managed from systemd
A nice value can also be defined in systemd unit files.

To check the nice value setting for a systemd unit, you can use the `systemctl show` command.

    systemctl show <name>.service | grep "Nice"

(To check the priority and nice value of an individual process, you can use the `ps -o pid,comm,pri,ni <PID>` command.) 

## Isolating resource consumption of PostgreSQL instances in systemd

Having looked at all of those settings in general, let's put them to use by implementing isolation of resource consumption for PostgreSQL in systemd. You will see how service units can be set up, and what to put into the unit files for those units for resource control purposes.

### Setting up systemd service units for PostgreSQL instances
Most Linux distributions have a default PostgreSQL service unit (like `postgresql.service`), for a simple setup, with a single PostgreSQL server instance. It usually looks something like this (this was taken almost word by word from SUSE Linux, but of course it may be slightly different with in distribution):

    root@sysagnostic:~ # systemctl cat postgresql.service

    # /usr/lib/systemd/system/postgresql.service
    [Unit]
    Description=PostgreSQL database server
    After=syslog.target
    After=network.target

    [Service]
    Type=forking
    User=postgres
    EnvironmentFile=-/etc/sysconfig/postgresql
    ExecStart=/usr/share/postgresql/postgresql-script start
    ExecStop=/usr/share/postgresql/postgresql-script stop
    ExecReload=/usr/share/postgresql/postgresql-script reload
    SendSIGKILL=no

    [Install]
    WantedBy=multi-user.target

You can then add resource control parameters into the `[Service]` section above, like this:
    
    [Service]
    Nice=-10
    Type=forking
    ...

A better way to add such parameters is to use the drop-in configuration mechanism in Systemd. You can create something called a *drop-in file*, and in that file you can define additional configuration for a unit (and overwrite existing configuration defined in the main unit file). In this case, you can create a file called `/etc/systemd/system/postgresql.service.d/override.conf` (or it is even better to use the `systemctl edit postgresql.service`) and provide your resource control settings in the `[Service]` section in that file.

That works just fine for a single PostgreSQL server instance.

When there are multiple instances, however, it makes sense to have a separate systemd service unit for each instance. Some Linux distributions will do it for you automatically (eg. in Debian you can use the [`pg_createcluster`](https://manpages.debian.org/bookworm/postgresql-common/pg_createcluster.1.en.html) to have a service unit created), while in other distributions you will have to create them yourself.

#### Template service and instantiated services
Systemd can handle something called a [*template service*](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html#Service%20Templates). A template service is a special service unit that can be instantiated. In our case, the the instantiated service units will serve as the actual service units for the different PostgreSQL server instances.

The name of the template service unit will be `postgresql-<ver>@.service` (where `<ver>` is the PostgreSQL major version). Notice the `@` at the end of the name. 

    root@sysagnostic:~ # cat /etc/systemd/system/postgresql-17@.service
    [Unit]
    Description=PostgreSQL database server instance %i
    After=syslog.target
    After=network.target

    [Service]
    Type=forking
    User=postgres
    EnvironmentFile=-/etc/sysconfig/postgresql
    Environment=POSTGRES_DATADIR=/var/lib/pgsql/17/%i/data
    ExecStart=/usr/share/postgresql/postgresql-script start
    ...

To instantiate that template, a certain value will need to go between `@` and `.service` in its name. In our case, a systemd service unit instances will be created for each PostgreSQL instance as `postgresql-<ver>@<cluster_name>.service`, where `<cluster_name>` is the name of the PostgreSQL server instance. 

PostgreSQL *cluster* is just an other name for PostgreSQL server instance, it has nothing to do with HA clusters. More precisely, it is the set of database a server instance is managing, typically located in the data directory of the PostgreSQL instance. 

What comes between `@` and `.service` in the service is also a variable, which is used inside the template. In our case it is the `<cluster_name>`, which will be used to point to the right data directory.

Notice the `%i` placeholder, that's where our variable value (the cluster name) will be substituted to, and will be used to specify the location of the data directory (in the `POSTGRES_DATADIR` environment variable, which will then be picked up by the `postgresql-script`; note that each distribution does this slighlty differently).

To instantiate that service unit, you can now simply add a cluster name (the data directory must exist) and start the new service:

    systemctl start postgresql-17@mypg1.service

To start an other instance, just pass another cluster name:

    systemctl start postgresql-17@mypg2.service


### Setting up resource control per PostgreSQL instance

Now that you have set up service units, you can add resource control parameters. But how? – you might wonder, because we only created a single, template unit file. 

Per-instance resource control settings can be provided in the `[Service]` section of a drop-in file: `/etc/systemd/system/postgresql-<ver>@<cluster_name>.service.d/99-resource-control.conf`. Note that the directory name ending in `service.d` has the exact name of the service unit as its first part. The file itself can be named anything, the important thing is that it ends in `.conf`.

The settings you define in that file will be picked up by systemd as drop-in configuration, and will merge it into the main configuration in the `[Service]` section of the service unit file.

You might have to run the `systemctl daemon-reload` command for systemd to load the file, and then restart the service using the `systemctl restart` command.

Let's now look at some useful resource control settings you can put into the drop-in file.

#### Limiting CPU usage 
You can use the [`CPUQuota`](https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html#CPUQuota=
) setting to limit CPU usage, which is a percentage value. The percentage specifies how much CPU time the unit can get at maximum, relative to the total CPU time. 

For example, to assign one full CPU to the instance, specify `CPUQuota=100%`. To assign 2 CPUs to the instance, specify `CPUQuota=200%`.

#### Limiting RAM usage
You can use the [`MemoryHigh`](https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html#MemoryHigh=) setting to limit RAM usage. 

For example, to allot 8GB of RAM to the instance, specify `MemoryHigh=8G`.

#### Prioritizing an instance
You can use the `Nice` setting, as shown above.

For example, to give more priority to you instance, specify `Nice=-15`. 
Priority can be set in the range of [-20, 19], and remember that a nicer process is lower priority.

#### Putting it all together
    root@sysagnostic:~ # cat/etc/systemd/system/postgresql-17@mypg1.service.d/99-resource-control.conf

    [Service]
    CPUQuota=400% ## 4 CPUs
    MemoryHigh=16G ## 16 GB RAM
    Nice=-15


    root@sysagnostic:~ # cat/etc/systemd/system/postgresql-17@mypg2.service.d/99-resource-control.conf

    [Service]
    CPUQuota=100% ## 1 CPU
    MemoryHigh=16G ## 8 GB RAM

