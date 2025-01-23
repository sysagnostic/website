---
draft: true
author: "Tamas Rebeli"
title: Isolating resource consumption for PostgreSQL instances in Linux
description: Overview of core Linux techniques, and using systemd to set resource limits per PostgreSQL instance
publishDate: 2025-01-15
date: 2024-01-08
Image_webp: images/blog/isolating-postgres-instances-cover.webp
image: images/blog/isolating-postgres-instances-cover.png
tags:
- Linux
- cgroups
- systemd
- resource control
- resource limits
- nice
---

In a traditional, on-premises enterprise environment, a single bare-metal server or virtual machine may host multiple PostgreSQL server instances. Even though they share the same machine, they often belong to different applications, projects or organizational units, and are required to be isolated in terms of utilizing system resources like RAM, CPU and disk I/O, usually for the purpose of business management.

PostgreSQL is an excellent database management system, but its built-in resource management – if you can even call it that – is mostly at a lower level than what would be ideal for setting up such an isolation in a straightforward manner. You can set some resource consumption related parameters to control things like memory utilization and degree of parallelism for certain SQL operations, but most of those settings do not impose overall hard limits for the PostgreSQL server instance (nor for databases, users or even queries for that matter). 

Setting up priorities between users, queries or databases so that one may consume more resources than others is a feature completely missing from community PostgreSQL too. Then again, even if prioriziation were possible at those levels, it is generally recommended to deploy a separate PostgreSQL server instance per database, unless a set of databases logically belong together, and may be managed (stopped and started, backed up, upgraded etc.) as a unit in terms of business functionality.

Does it all mean such isolation is impossible? No.

> It is possible to isolate PostgreSQL instances, but as with quite a few other aspects in PostgreSQL, limiting overall resource consumption or setting up priorities needs to be implemented at the operating system level.


After a run-down of some basic resource control techniques in Linux, we are going to cover how resource management is made possible from systemd, and how to do it specifically for PostgreSQL.


## Core Linux technologies for resource management
Sharing resources between users on a server has always been the norm from the very early ages of computing, and controlling resource consumption came as a natural requirement. Some of the older tools discussed here date back to the 1970's, but even the most recent core technology originated in the late 2000's. That means you've probably heard about all of these. Let's take a quick look.

### Classical resource limits
What first comes to mind is the age-old resource limiting solution commonly referred to as [ulimit](https://pubs.opengroup.org/onlinepubs/9699919799/functions/ulimit.html). POSIX originally used this name for process limits, but later deprecated it in favour of the new name *rlimit* (resource limit).

In GNU/Linux, corresponding system calls are also implemented by [functions](https://www.gnu.org/software/libc/manual/html_node/Limits-on-Resources.html) with **`rlimit`** in their names, like the [`prlimit()`](https://man7.org/linux/man-pages/man2/prlimit.2.html) system call,
which can set resource limits per process. The shell built-in to get and set resource limits on the command line in an OS user session (for child processes forked by the shell) somehow kept the old name `ulimit` in Linux, but the command to change limits for already running processes is [`prlimit`](https://man7.org/linux/man-pages/man1/prlimit.1.html), following the new naming.

Linux also has something called *security limits*, which is basically a [PAM](https://www.man7.org/linux/man-pages/man3/pam.3.html) wrapper around *rlimit* calls to automate `ulimit` for OS user session creation, traditionally configured in [`/etc/security/limits.conf`](https://www.man7.org/linux/man-pages/man5/limits.conf.5.html).

With classical resource limits, you can set limits on individual processes in a given OS user session. They are not per-user settings, they are per-process setting: Linux does not sum resource consumption over processes for a user. Child processes inherit the limits set on the parent. Limits apply to resources like the maximum CPU time in seconds the process can use (`-t` or `cpu`), the maximum virtual memory size (address space) (`-v` or `as`), and the maximum stack size (`-s` or `stack`) the process can allocate. Please note that limiting physical memory usage (resident set size) (`-m` or `rss`) was disabled a long time ago, in kernel 2.4.30. Setting that limit will have no effect. 

You might face the need to change some of these settings every now and then (for example: increase the stack size to allow a higher value for the `max_stack_depth` PostgreSQL server parameter). Classical resource limit settings, however, are not instrumental to isolating PostgreSQL instances. Limiting CPU time is obviously not a great idea, nor is limiting virtual memory of a process instead of actual RAM usage, and RSS size cannot be limited. There is no way to limit I/O usage either.


### Nice value
As for setting priorities, another long-standing POSIX concept called [*nice value*](https://pubs.opengroup.org/onlinepubs/9799919799/functions/nice.html) may come to mind.

A "nicer" process consumes less resources, which means increasing niceness will make the process lower priority, or in other words the CPU scheduler will favour the process less. Child processes will inherit their priority and nice values from their parents.

In GNU/Linux, you can use the [`nice`](https://man7.org/linux/man-pages/man1/nice.1.html) command to run a program with a specified nice value, and the [`renice`](https://man7.org/linux/man-pages/man1/renice.1.html) command to interactively change the nice value for a given process, while the same can be achieved programmatically with the [`setpriority()`](https://man7.org/linux/man-pages/man2/setpriority.2.html) system call. 

You can increase the priority (set a negative nice value) for your own processes only as root, but you can decrease the priority (set a positive nice value), although only up to the configured limit (see below). The new priority will be calculated as: *current priority + nice value*. 

Unfortunately, the terms "nice value" and "priority" are sometimes used interchangeably, which can get quite confusing. For example, the `renice` command refers to the nice value as "priority", and in classical resource limits, the two terms are even more conflated.

Classical resource limits also include limits for both nice value and priority, to be applied at the user session level. With `ulimit`, you can set a limit called "scheduling priority" (`-e`), which indeed limits the maximum priority. So far so good. In `limits.conf`, however, there are two limits: `priority` and `nice`. When a hard limit for `priority` is set, it is interpreted and set as the nice value for processes directly, which in turn affects priority as usual. When a hard limit for `nice` is set, it will change the scheduling priority limit (`-e`), and will be the maximum priority the user can set for its processes.

Adjusting the nice value for a client backend process of a PostgreSQL server instance is a feasible method to give priority to a given database session. You can also adjust the nice value of the `postgres` process (which is the parent of all other processes in the PostgreSQL instance) to prioritize that particular instance over others. 

Prioritizing disk I/O for processes is also possible in Linux via [`ionice`](https://man7.org/linux/man-pages/man1/ionice.1.html) on the command line, and via the [`ioprio_set()`](https://man7.org/linux/man-pages/man2/ioprio_set.2.html) system call programmatically, but it requires the [CFQ scheduler]( https://www.kernel.org/doc/Documentation/block/cfq-iosched.txt) to be set for disk devices used by PostgreSQL, which may not be the default I/O scheduler in your distribution of choice, and might just not be the best option for your particular workload.


### Control groups
A more recent and more featureful resource management solution implemented in the Linux kernel is *cgroups* ([Control Groups](https://docs.kernel.org/admin-guide/cgroup-v1/cgroups.html)), originally developed by Google engineers, which ran by the name *Process Containers* at that time (and indeed it is one of the main pillars of today's container technology in Linux, along with namespaces).

Cgroups allows processes to be organized into a hierarchy of groups, and provides control over resource consumption for those groups of processes. 
Every process inside a cgroup will have the limits of that cgroup imposed on it. 

Different controllers are available in cgroups to limit the consumption of different types of resources (CPU, memory, block I/O etc.). These controllers provide much more fine-grained control over resource consumption than earlier solutions.

In 2015, a new version of cgroups, [*cgroups-v2*](https://docs.kernel.org/admin-guide/cgroup-v2.html) made its way into the Linux kernel (in version 4.5). It is also commonly referred to as the *unified hierarchy*, because that was a major change from v1 (in v1, every controller had a different mount point under `/sys/fs/cgroup/`), but other changes were introduced as well, including changes that affect I/O and memory control. Newer Linux distributions are using cgroups-v2 now, but you may very well see Linux installations with the original cgroups still.


## Using systemd for resource management
You can use those core Linux technologies introduced above in a more centralized (and perhaps less confusing) manner by an even more recent technology, [*systemd*](https://man7.org/linux/man-pages/man1/init.1.html). It is an init system and service manager for GNU/Linux, originally developed by Red Hat, and later adopted by all major Linux distributions, following long and heated debates. Systemd is now wide-spread, and you must get familiar with it if you haven't done so.

This article will focus on resource management solutions based on systemd. But just before we get to that, let's briefly look at how systemd organizes processes that it is managing.


### Systemd unit types
Systemd manages different types of units. Important types for the purpose of this article are: *service*, *scope* and *slice*.

#### Services and scopes
You are most probably familiar with service units. A *service* runs one or more processes, or in other words, a service is a logical group of processes. A service unit is defined in its [*unit file*](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html) `<name>.service`.

A *scope* is created automatically by systemd for each user session (and some other things) running in Linux. A scope is also a logical group of processes. (The difference is that the processes in a scope are not started directly by systemd, they are merely assigned to the scope by systemd. That is also why you cannot define a scope in a unit file.)

You can use the `systemctl --type scope` command to list scopes.

    root@sysagnostic:~# systemctl --type scope
    UNIT                 LOAD   ACTIVE SUB     DESCRIPTION
    init.scope           loaded active running System and Service Manager
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


### Cgroups managed from systemd
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

(You can use the `systemctl status` command for a unit to see where it is located in the cgroups hierarchy.)

As you might have guessed, the cgroups hierarchy maintained by systemd allows you to define cgroups resource limits for slices, services and scopes. Such definitions go into the corresponding unit files (`<name>.slice` or `<name>.service`), or limits can be set interactively for an already runing unit in a dynamic fashion (that is the only option for scopes).

Available parameters related to CPU, RAM and I/O resource control are all explained in the [Resource control unit settings](https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html) page of the systemd documentation. 

You can use the `systemctl show` command to see the current settings for a unit, including the ones related to resource control.

    systemctl show <name>.service | grep -E "^CPU|^Memory|^IO"

We will come back to some specific resource control settings shortly. Before we do that, let's touch upon the older limiting techniques once more, to see how they integrate into systemd.


### Classical resource limits managed from systemd
A systemd service unit can have rlimit (ulimit) parameters defined for it, which apply to all of the processes included in that unit. 
There is a [mapping between ulimit and systemd parameter names](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#id-1.12.2.1.17.6).

To check rlimit settings for a systemd unit, you can use the `systemctl show` command.

    systemctl show <name>.service | grep "^Limit"

(To check rlimit settings for an individual process, you can read the `/proc/<PID>/limits` file.)

### Nice value managed from systemd
A nice value can also be defined in systemd service unit files.

To check the nice value setting for a systemd unit, you can use the `systemctl show` command.

    systemctl show <name>.service | grep "Nice"

(To check the priority and nice value of an individual process, you can use the `ps -o pid,comm,pri,ni -q <PID>` command.) 

## Isolating resource consumption of PostgreSQL instances in systemd

Having looked at all of those settings in general, let's put them to use by implementing isolation of resource consumption for PostgreSQL in systemd. You will see how service units can be set up, and what to put into the unit files for those units for resource control purposes.

### Setting up systemd service units for PostgreSQL instances
Most Linux distributions have a default PostgreSQL service unit (like `postgresql.service`), for a simple setup, with a single PostgreSQL server instance. It usually looks something like this (this was taken almost word by word from SUSE Linux, but of course it will be slightly different with each distribution):

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

You can add resource control parameters into the `[Service]` section above, like this:

    [Service]
    Nice=-10
    Type=forking
    …

A better way to add such parameters is to use the drop-in configuration mechanism in Systemd. You can create something called a *drop-in file*, and in that file you can define additional configuration for a unit (and overwrite existing configuration defined in the main unit file). In this case, you can create a file called `/etc/systemd/system/postgresql.service.d/override.conf` (or it is even better to use `systemctl edit postgresql.service`) and provide your resource control settings in the `[Service]`.

That works just fine for a single PostgreSQL server instance.

> When there are multiple instances, however, it makes sense to have a separate systemd service unit for each PostgreSQL server instance. 

Some Linux distributions will do it for you automatically (for example in Debian, you can use [`pg_createcluster`](https://manpages.debian.org/bookworm/postgresql-common/pg_createcluster.1.en.html) to have a service unit created), while in other distributions you will have to create the service units yourself.

#### Template service and instantiated services
Systemd can handle something called a [*template service*](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html#Service%20Templates). A template service is a special service unit that can be instantiated. In our case, the instantiated service units will serve as the actual service units for the different PostgreSQL server instances.

The name of the template service unit will be `postgresql-<ver>@.service` (where `<ver>` is the PostgreSQL major version). Notice `@` in the name. Here is what the unit file could look like: 

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
    …

To instantiate a template, a certain value will need to go between `@` and `.service` in its name. In our case, a systemd service unit instance will be created for each PostgreSQL instance as `postgresql-<ver>@<cluster_name>.service`, where `<cluster_name>` is the name of the PostgreSQL server instance. 

(PostgreSQL *cluster* is just another name for PostgreSQL server instance, it has nothing to do with HA clusters. More precisely, a culster is the set of databases that a PostgreSQL server instance is managing, typically located in the data directory.)

What comes between `@` and `.service` is also a variable, which is used inside the template. In our case, it is the `<cluster_name>`, which will be used to point to the data directory of the given cluster.

Notice the `%i` placeholder. It is where our variable value (the `<cluster_name>`) will be substituted. It will be used to specify the location of the data directory (in the `POSTGRES_DATADIR` environment variable, which will then be picked up by the `postgresql-script` – please note that each distribution does this slightly differently).

To instantiate that service unit, you can now simply add a cluster name (the data directory must exist), and start the new service:

    systemctl start postgresql-17@mypg1.service

To start another instance, just pass another cluster name:

    systemctl start postgresql-17@mypg2.service


### Setting up resource control per PostgreSQL instance

Now that you have set up service units, you can add resource control parameters. "But how?", you might wonder, because you've only created a single template unit file. 

Per-instance resource control settings can be provided in the `[Service]` section of a drop-in file, as discussed before. In our case, the drop-in file will be `/etc/systemd/system/postgresql-<ver>@<cluster_name>.service.d/99-resource-control.conf`. Notice that the directory name ending in `service.d` must begin by the exact name of the service unit. The file itself can be named anything, the important thing is that it ends in `.conf`.

The settings you define in the file will be picked up by systemd as drop-in configuration, and will merge it into the main configuration.

You might have to run the `systemctl daemon-reload` command for systemd to load the file, and then restart the service using the `systemctl restart` command.

Let's now look at some useful resource control settings you can put into the drop-in file.

#### Limiting CPU usage 
You can use the [`CPUQuota`](https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html#CPUQuota=
) setting to limit CPU usage, which is a percentage value. The percentage specifies how much CPU time the unit can get at maximum, relative to the total CPU time. 

For example, to assign one full CPU to the instance, specify `CPUQuota=100%`. To assign 2 CPUs to the instance, specify `CPUQuota=200%`.

> This parameter will affect the `CPUQuotaPerSecUSec` attribute of the systemd service unit, which shows how many CPU seconds the process will get for 1 wall-clock second. For  example, for `50%` it will show as `500ms`, and for `200%` as `2s`.

#### Limiting RAM usage
You can use the [`MemoryHigh`](https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html#MemoryHigh=) setting to limit RAM usage. 

For example, to allot 8GB of RAM to the instance, specify `MemoryHigh=8G`.

> It is not a hard limit, which means the service unit may consume more memory, and it won't be killed. When memory consumption goes beyond that limit, however, the processes within the service unit are slowed down, and memory is taken away from them aggressively.

#### Prioritizing an instance
You can use the `Nice` setting, as shown above.

For example, to give more priority to you instance, specify `Nice=-15`. 
Priority can be set in the range of -20, 19, and please remember that a nicer process is lower priority.

#### Putting it all together
Set up one service:

    root@sysagnostic:~ # cat /etc/systemd/system/postgresql-17@mypg1.service.d/99-resource-control.conf

    [Service]
    ## max. 4 CPU
    CPUQuota=400%

    ## max. 16 GB RAM
    MemoryHigh=16G

    ## higher priority
    Nice=-15

Check settings:

    root@sysagnostic:~ # systemctl show postgresql-17@mypg1.service | grep -E "CPUQuotaPerSecUSec|MemoryHigh|Nice"
    CPUQuotaPerSecUSec=4s
    MemoryHigh=17179869184
    Nice=-6

Set up a second service:

    root@sysagnostic:~ # cat /etc/systemd/system/postgresql-17@mypg2.service.d/99-resource-control.conf

    [Service]
    ## max. 1 CPU
    CPUQuota=100%

    ## max. 8GB RAM
    MemoryHigh=8G

Check settings for the second service:

    root@sysagnostic:~ # systemctl show postgresql-17@mypg2.service | grep -E "CPUQuotaPerSecUSec|MemoryHigh"

    CPUQuotaPerSecUSec=1s
    MemoryHigh=8589934592

### Setting up resource control for a group of PostgreSQL instances 
Systemd automatically assigns instantiated service units to a slice unit that is named after their template unit. In our case, all PostgreSQL service units will be assigned to a slice unit called `system-postgresql.slice`. 

    root@sysagnostic:~ # systemd-cgls --unit system-postgresql.slice

    Unit system-postgresql.slice (/system.slice/system-postgresql.slice):
    system-postgresql.slice 
    ├─postgresql-17@mypg1.service 
    │ ├─606 /usr/lib/postgresql/17/bin/postgres -D /var/lib/postgresql/17/data/mypg1
    │ ├─669 postgres: mypg1: checkpointer
    │ ├─671 postgres: mypg1: background writer
    │ ├─673 postgres: mypg1: walwriter
    │ ├─675 postgres: mypg1: autovacuum launcher
    │ ├─677 postgres: mypg1: stats collector
    │ └─679 postgres: mypg1: logical replication launcher
    └─postgresql-17@mypg2.service 
      ├─607 /usr/lib/postgresql/17/bin/postgres -D /var/lib/postgresql/17/data/mypg2
      ├─709 postgres: mypg2: checkpointer
      ├─710 postgres: mypg2: background writer
      ├─711 postgres: mypg2: walwriter
      ├─712 postgres: mypg2: autovacuum launcher
      ├─713 postgres: mypg2: stats collector
      └─714 postgres: mypg2: logical replication launcher

You can configure resource control for a slice by creating a unit file for the slice unit. Settings defined in that unit file (in the `[Slice]` section) will apply to all the PostgreSQL instances it contains, regardless of the settings for each instance.

You can also set up your own slices. To define a slice, create a unit file for it. To assign a service to a slice, set `slice=<name>.slice` in the `[Service]` section.
