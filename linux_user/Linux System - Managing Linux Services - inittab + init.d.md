# Linux System - Managing Linux Services - inittab + init.d

## 1. The /etc/inittab File

When init starts up, it reads the **/etc/inittab** configuration file. While the system is running, it will re-read it, if sent the HUP signal (kill -HUP 1); this feature makes it unnecessary to boot the system in order to make changes to the init configuration take effect.

The **/etc/inittab** file is a bit complicated. We'll start with the simple case of configuring getty lines. Lines in /etc/inittab consist of four colon-delimited fields:

```
id:runlevels:action:process
```

The fields are described below. In addition, **/etc/inittab** can contain empty lines, and lines that begin with a number sign (`#'); these are both ignored.

* **id** - This identifies the line in the file. For getty lines, it specifies the terminal it runs on (the characters after /dev/tty in the device file name). For other lines, it doesn't matter (except for length restrictions), but it should be unique.
* **runlevels** - The run levels the line should be considered for. The run levels are given as single digits, without delimiters. (Run levels are described in the next section.)
* **action** - What action should be taken by the line, e.g., respawn to run the command in the next field again, when it exits, or once to run it just once.
* **process** - The Linux command to run.

```BASH
root@MiWiFi-RA67-srv:/etc# cat inittab
# /etc/inittab: init(8) configuration.
# $Id: inittab,v 1.91 2002/01/25 13:35:21 miquels Exp $

# The default runlevel.
id:5:initdefault:

# Boot-time system configuration/initialization script.
# This is run first except when booting in emergency (-b) mode.
si::sysinit:/etc/init.d/rcS

# What to do in single-user mode.
~~:S:wait:/sbin/sulogin

# /etc/init.d executes the S and K scripts upon change
# of runlevel.
#
# Runlevel 0 is halt.
# Runlevel 1 is single-user.
# Runlevels 2-5 are multi-user.
# Runlevel 6 is reboot.

l0:0:wait:/etc/init.d/rc 0
l1:1:wait:/etc/init.d/rc 1
l2:2:wait:/etc/init.d/rc 2
l3:3:wait:/etc/init.d/rc 3
l4:4:wait:/etc/init.d/rc 4
l5:5:wait:/etc/init.d/rc 5
l6:6:wait:/etc/init.d/rc 6
# Normally not reached, but fallthrough in case of emergency.
z6:6:respawn:/sbin/sulogin
#mxc0:12345:respawn:/bin/start_getty 115200 ttymxc0
mxc0:12345:respawn:/sbin/getty -l /bin/autologin -n -L 115200 ttymxc0
# /sbin/getty invocations for the runlevels.
#
# The "id" field MUST be the same as the last
# characters of the device (after "tty").
#
# Format:
#  <id>:<runlevels>:<action>:<process>
#

1:12345:respawn:/sbin/getty 38400 tty1
```

## What is init.d?

All these services work on several scripts and these scripts are stored in **/etc/init.d**. **init** is a daemon which is the first process of the Linux system. Then other processes, services, and daemons, are started by **init**. So init.d is a configuration database for the **init** process.

There are numerous files in the /etc/init.d directory. Each of these files represent a service, that is a daemon, that may or may not be running on the system. The files are simply Linux shell scripts that can be run either by **init**, or manually from the command line. The scripts all have some portions that are the same. If the service is to be started the "start" function is called, if it is to be stopped, then the "stop" function is called. So, each script has several functions to respond appropriately to the various sub-commands of service: start, stop, status, etc.

## The /etc/init.d Directory

In Linux there are several services that can be started and stopped manually in the system. Some of these services are ssh, HTTP, tor, apache, etc. To start and run these services we used to simply type **service "service name" start/stop/status/restart.**

**Example:**

`service ssh start`

And to check if this service is running we type the command:

`service ssh status`

In this simple manner, we are using service management in Linux but what actually happens and how it actually works is in the background.

Some documentation sources suggest that administrators run the scripts directly as in:

`/etc/init.d/ssh start`

However, it is better to use the **service** command, since it knows where the scripts are located, and it is less typing.

```
ID  Name                               Description
0   Halt                               Shuts down the system.
1   Single-user Mode                   Mode for administrative tasks.
2   Multi-user Mode                    Does not configure network interfaces and 
                                       does not export networks services.
3   Multi-user Mode with Networking    Starts the system normally.
4   Not used/User-definable            For special purposes.
5   Start the system normally with 
    with GUI                           As runlevel 3 + display manager.
6   Reboot                             Reboots the system.
```

The content of the rc* dir:

![](https://raw.githubusercontent.com/carloscn/images/main/typora20230114132953.png)


## The /etc/rc.local Directory

There is also a directory /etc/rc.local. It starts to get confusing as to what are all these directories and files for.

-   **init.d** is the directory that stores services control scripts, which control the starting and stopping of services such as httpd, sshd, etc.
-   **rc.local** is a service that allows running of arbitrary scripts as part of the system startup process.

So, if you have local scripts that you as the administrator want to run when the system is started, you can place them in the /etc/rc.local directory and they get run automatically.

## The **service** Command

The **service** command is used to run a System V init script. Usually all system V init scripts are stored in /etc/init.d directory and service commands can be used to start, stop, and restart the daemons and other services under Linux. All scripts in /etc/init.d accept and support at least the start, stop, and restart commands.

Syntax:

`service SCRIPT COMMAND [ OPTIONS ]`

The SCRIPT is the name of one the scripts that exists in /etc/init.d. The allowable COMMAND parameters depends on the SCRIPT, but the scripts should all at least have a "start" and a "stop" capability. The COMMAND and OPTION parameters are passed directly to the script itself, where they are interpreted. In the following table each row shows the service command, "name," which is the service that is being queried (such as httpd orsshd), and then the sub-command.

| **Sub-commands**         | **Meaning**                         |
| ------------------------ | ----------------------------------- |
| service name **start**   | Starts a service                    |
| service name **stop**    | Stops a service                     |
| service name **restart** | Restarts a service                  |
| service name **reload**  | Reloads a configuration             |
| service name **status**  | Checks whether a service is running |
| service –status- all     | Displays the status of all services |

## The **chkconfig** Command

The **chkconfig** command is quite similar to the **service** command, and is used to list all available services and view or update their run level settings. It lists current startup information of services or any particular service, updating runlevel settings of service and adding or removing service from management.

Syntax:

`chkconfig [ OPTIONS ] [ SERVICE ]  sub-command`

In the following table the only OPTION used is --list, though there are several other options - consult your **chkcofig** man page for additional information. In these examples, replace SERVICE with the appropriate service name.

| **Sub-command**                | **Meaning**                                                  |
| ------------------------------ | ------------------------------------------------------------ |
| chkconfig SERVICE on           | Enables a service                                            |
| chkconfig SERVICE off          | Disables a service                                           |
| chkconfig –list SERVICE        | Checks to see if a service is enabled                        |
| chkconfig –list                | List all services and check if they are enabled              |
| chkconfig SERVICE reset        | Restore the service’s startup priority and other information to the recommended settings |
| chkconfig –level 5 SERVICE on  | Enable the specified service at the specified runlevel (in this case runlevel 5) |
| chkconfig –level 5 SERVICE off | Disable the specified service at the specified runlevel (in this case runlevel 5) |


[^1]

# Ref
[^1]:[08-C.8.4: Managing Linux Services - inittab & init.d](https://eng.libretexts.org/Bookshelves/Computer_Science/Operating_Systems/Linux_-_The_Penguin_Marches_On_(McClanahan)/08%3A_How_to_Manage_System_Components/3.08%3A_Managing_Linux_Services/3.08.04%3A_Managing_Linux_Services_inittab_init.d)