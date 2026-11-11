# friByte systemd workshop 2026
This is a workshop intended to teach the basics of working with systemd on a Linux based system.

> This workshop assumes the use of an Ubuntu Server LTS 26.04.x system, but is mostly applicable to other distributions aswell. Depending on the distribution there might be a difference in the preinstalled systemd units on the system.

# Workshop

## Setting up a system to complete the workshop

If you are an active member of friByte and have access to our internal infrastructure you are encouraged to create a new VM in Proxmox using the instructions found in our [public wiki [1]](#sources).

Others are also encouraged to participate in the workshop with their own system, either on a virtual machine or bare metal. Instructions can be found in the [Ubuntu documentation [2]](#sources).

## What is systemd
Systemd is an init system and service manager developed for the Linux ecosystem. This means that many of the running services and or operational tasks of a Linux system are already or can be managed using systemd. This can also be expanded upon by creating your own custom services that are managed by systemd. Which is quite handy! Systemd also offers a handful of other components in it's ecosystem that provide system services like logging through journald, system locales through localed, network interfaces through networkd and many other essential services

## Preinstalled Systemd services and the systemctl CLI

Systemd runs background processes and one off tasks called units. These units can be one of several types. See the table below for each unit file type and their function [[3]](#sources).

| Unit file | Unit function                                              |
|-----------|------------------------------------------------------------|
| .service  | A process or daemon                                        |
| .timer    | Schedule that activates another unit                       |
| .socket   | Listening socket that starts a service on demand           |
| .target   | Group of units, used as a boot milestone                   |
| .path     | Watches a file or directory and activates a unit on change |
| .mount    | Filesystem mount point                                     |
| .slice    | Resource control group for other units                     |

Let's first look at what units are running on the system already. For this we will use the `systemctl` CLI

```console
systemctl list-units
```

This shows us all systemd-managed units currently running on the system. Now this can be quite a lot as we now have a list of all currently running unit types. To get a sense of only the service units on the system, i.e background programs running we instead list only the units of type service with the following command.

```console
systemctl list-units --type service
```

Now we have a list of the running services, which is still a lot but some of the names should look familiar. Let's take a look at `ssh.service`.

> This command requires privelege escalation with sudo in order to also read the system log files related to the service. (Which are also managed by systemd btw, but we will look more into this later)

```console
sudo systemctl status ssh.service
```

Produces an output similar to this:

```console
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/usr/lib/systemd/system/ssh.service; disabled; preset: enabled)
     Active: active (running) since Sun 2026-11-08 13:56:52 UTC; 1 day 4h ago
 Invocation: 230fa3eb5d96499fb6448f2663a0e5d2
TriggeredBy: ● ssh.socket
       Docs: man:sshd(8)
             man:sshd_config(5)
   Main PID: 1015 (sshd)
      Tasks: 1 (limit: 3248)
     Memory: 7.6M (peak: 9.7M)
        CPU: 215ms
     CGroup: /system.slice/ssh.service
             └─1015 "sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups"
```

Let's look at what information this output gives us. 
- First of all we see that this is a service for the OpenBSD Secure Shell Server.

- The service has loaded from the service file `/usr/lib/systemd/system/ssh.service`. This is the service unit file which defines the behaviour of the service, we will look at the content of this file later.

- The service is active with a state of running and the service has been running for a little over one day.

- The service was triggered by the `ssh.socket` systemd unit, this means that the service is a so called "socket activated service" and did not start by itself but was instead told to start by the accompanying ssh socket which triggers when the machine is reached from the bound network port of `22`.

- Docs shows the associated documentaion which in this case refers to the manpage pages of the program and it's configuration

- Main PID is the Process ID of the sshd process started by the service unit. In this case a process ID of 1015. We can confirm this by running `ps 1015` which shows this is an active process on the system with PID 1015

- Memory shows how much memory is currently used by the service

- CGroup shows what cgroup the process belongs to. This is a group of processes that can have their resources shared. In this case there is only the main process in the cgroup

- The bottom of the output shows the latest logs of the service as long as the command is run as root

## Journald logging and the journalctl CLI

Journald is a part of systemd which manages the internal system log as well as supporting logging for systemd services. This means that most of the logs written by running services/processes are kept by journald and can be managed using the `journalctl` CLI. The broadest use of this command is simply viewing all current logs at once.

```console
sudo journalctl --follow
```

This command will show logs written in real time since we supplied the `--follow` (shorthand `-f`) argument. Since we did not supply any specific unit to monitor journalctl shows all the logs on the system written to journald. Now let's look at one specific unit instead.

```console
sudo journalctl --unit ssh.service
```
This shows us a scrollable page of logs related to the sshd service where `--unit ssh.service` is an argument specifying the unit to list the logs of. Now you might notice these logs could be a bit out of date, this is because currently we are viewing logs from the beginning. Usually we are most interrested in what is either being logged right now or has been logged recently. Let's look at a more useful version of this command.

```console
sudo journalctl -xeu sshd.service --since today
```

This will show us the most recent logs of the sshd service today. Now let's break down the arguments used here. `-xeu` is actually a combination of the arguments `-x`, `-e` and `-u`. This could also be written as `--catalog --pager-end --unit ssh.service`. While the last argument `--since` lets us view only logs within a certain timeframe, from today in this case.

## Managing the service of a newly installed program

Up until now we have looked at services preinstalled on the system, but what if we want to install and run something else that is not already installed on the system. Let's use the Nginx webserver and proxy program as an example. First we want to install nginx from our distribution package repository.

```console
sudo apt install nginx
```

Now doing this installs the Nginx executable on the system aswell along with it's configuration files. In addition to this the `nginx-common` sub-package dependency also installs an nginx service unit definition on the system. This file will be located at the location of `/usr/lib/systemd/system/nginx.service` on the system [[4]](#sources). Let's look at the contents of this unit file.

```console
fribyte@systemd-workshop:~$ cat /usr/lib/systemd/system/nginx.service

[Unit]
Description=nginx - high performance web server
Documentation=https://nginx.org/en/docs/
After=network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/run/nginx.pid
Environment="CONFFILE=/etc/nginx/nginx.conf"
Environment="cmdopts="
EnvironmentFile=-/etc/default/nginx
ExecStart=/usr/sbin/nginx -c ${CONFFILE} $cmdopts
ExecReload=/bin/sh -c "/bin/kill -s HUP $(/bin/cat /run/nginx.pid)"
ExecStop=/bin/sh -c "/bin/kill -s TERM $(/bin/cat /run/nginx.pid)"

[Install]
WantedBy=multi-user.target
```

- Show status, config, enabled, disabled state, stop, start, restart, daemon reload

## Making our own service

- Make a simple service around a simple program
- Maybe socket activated??
- https://deepwiki.com/codyps/rust-systemd/4.1-socket-activation

## Other types of systemd units

- Sockets, mounts etc

# Sources

- [1] VM setup instructions https://wiki.fribyte.no/docs/instrukser/ny-vm/ (accessed 09.11.2026)
- [2] Ubuntu Server tutorial https://ubuntu.com/server/docs/tutorial/ (accessed 09.11.2026)
- [3] systemd Unit File Cheatsheet https://linuxize.com/cheatsheet/systemd-unit-file/ (accessed 10.11.2026)
- [4] File list of package nginx-common in resolute of architecture all https://packages.ubuntu.com/resolute/all/nginx-common/filelist (accessed 11.11.2026)
