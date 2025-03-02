---
draft: true
author: "Tamas Rebeli"
title: Isolating resource consumption for PostgreSQL instances in Linux – Part 2
description: Overview of core Linux techniques, and using systemd to set resource limits per PostgreSQL instance
publishDate: 2025-02-15
date: 2024-02-15
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
Having looked at the general resource limit settings in [Part 1](/blog/isolating-postgres-instances-p1), let's put them to use by configuring isolation of resource consumption for PostgreSQL via systemd.

## Configuring resource consumption isolation of PostgreSQL instances via systemd

We will take a look at how service units can be set up, and what unit file parameters may be used for resource control purposes.

### Setting up systemd service units for PostgreSQL instances
Most Linux distributions have a default PostgreSQL service unit (like `postgresql.service`), for a simple setup, with a single PostgreSQL server instance. The example unit file was copied from SuSE Linux distribution, but of course it can be slightly different on each distribution:

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

A better way to add such parameters is to use the drop-in configuration mechanism in Systemd. You can create something called a *drop-in file*, and in that file you can define additional configuration for a unit and merge your configuration with the existing one defined in the main unit file. In this case, you can create a file called `/etc/systemd/system/postgresql.service.d/override.conf` - or use the equivalent command `systemctl edit postgresql.service` - and provide your resource control settings in the `[Service]` section.

> When there are multiple instances, however, it makes sense to have a separate systemd service unit for each PostgreSQL server instance.

Some Linux distributions will do it for you automatically. For example, in Debian, you can use [`pg_createcluster`](https://manpages.debian.org/bookworm/postgresql-common/pg_createcluster.1.en.html) to have a service unit created, while in other distributions you have to create the service units yourself.

#### Template service and instantiated services
Systemd can handle something called a [*template service*](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html#Service%20Templates). A template service is a special service unit that can be instantiated. In our case, the instantiated service units will serve as the actual service units for the different PostgreSQL server instances.

The name of the template service unit will be `postgresql-<ver>@.service`, where `<ver>` is the PostgreSQL major version. Notice `@` in the name. Here is what the unit file could look like: 

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

Note that PostgreSQL *cluster* is just another name for PostgreSQL server instance, it has nothing to do with HA clusters. More precisely, a culster is the set of databases that a PostgreSQL server instance is managing, typically located in the data directory.

What comes between `@` and `.service` is also a variable, which is used inside the template. In our case, it is the `<cluster_name>`, which will be used to point to the data directory of the given cluster.

Notice the `%i` placeholder in the unit descriptor. The `%i` placeholder is where our variable value (the `<cluster_name>`) will be substituted. It specifies the location of the data directory. In our case, the data directory is defined in the `POSTGRES_DATADIR` environment variable, which is passed to the `postgresql-script` when PostgreSQL is started. Please note that each distribution does this slightly differently.  

You can now simply instantiate a service unit and start the new service by passing the cluster name to systemctl. Please note, that the data directory must exist before starting the service.

    systemctl start postgresql-17@mypg1.service

To start another instance, just pass another cluster name:

    systemctl start postgresql-17@mypg2.service


### Setting up resource control per PostgreSQL instance

Now that you have set up service units, you can add resource control parameters. "But how?", you might wonder, because you've only created a single template unit file. 

Per-instance resource control settings can be provided in the `[Service]` section of a drop-in file, as discussed before. In our case, the drop-in file will be `/etc/systemd/system/postgresql-<ver>@<cluster_name>.service.d/99-resource-control.conf`. Notice that the directory name ending in `service.d` must begin by the exact name of the service unit. The file itself can be named anything, the important thing is that it ends in `.conf`.

The settings you define in the file will be picked up by systemd as drop-in, and will merge it into its main configuration.

You need to run the `systemctl daemon-reload` command after changing systemd unit files or drop-ins to reload the new definitions. After a successful reload, restart the service using the `systemctl restart` command.

Let's now look at some useful resource control settings you can put into the drop-in files.

#### Limiting CPU usage 
You can use the [`CPUQuota`](https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html#CPUQuota=
) setting to limit CPU usage, which is a percentage value. The percentage specifies how much CPU time the unit can get at maximum, relative to the total CPU time available to the operating system for scheduling.

For example, to assign one full CPU to the instance, specify `CPUQuota=100%`. To assign 2 CPUs to the instance, specify `CPUQuota=200%`.

> This parameter will affect the `CPUQuotaPerSecUSec` attribute of the systemd service unit, which shows how many CPU seconds the process will get for 1 wall-clock second. For  example, for `50%` it will show as `500ms`, and for `200%` as `2s`.

#### Limiting RAM usage
You can use the [`MemoryHigh`](https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html#MemoryHigh=bytes) setting to limit RAM usage.

For example, to allot 8GiB of RAM to the instance, specify `MemoryHigh=8G`.

> It is not a hard limit, which means the service unit may consume more memory, and it won't be killed. When memory consumption goes beyond that limit, however, the processes within the service unit are slowed down, and memory is taken away from them aggressively.

#### Prioritizing an instance
You can use the `Nice` setting, as shown above.

For example, to give more priority to your instance, specify `Nice=-15`.
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

    root@sysagnostic:~ # systemctl show postgresql-17@mypg1.service --property=CPUQuotaPerSecUSec,MemoryHigh,Nice
    CPUQuotaPerSecUSec=4s
    MemoryHigh=17179869184
    Nice=-15

Set up a second service:

    root@sysagnostic:~ # cat /etc/systemd/system/postgresql-17@mypg2.service.d/99-resource-control.conf

    [Service]
    ## max. 1 CPU
    CPUQuota=100%

    ## max. 8GB RAM
    MemoryHigh=8G

Check settings for the second service:

    root@sysagnostic:~ # systemctl show postgresql-17@mypg2.service --property=CPUQuotaPerSecUSec,MemoryHigh,Nice

    CPUQuotaPerSecUSec=1s
    MemoryHigh=8589934592
    Nice=0

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
