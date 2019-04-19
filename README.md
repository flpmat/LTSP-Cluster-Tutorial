# LTSP-Cluster (Tutorial)

LTSP-Cluster is a set of LTSP plugins and client-side tools that allows you to deploy and centrally manage large numbers of thin-clients. It allows you to run thousands of thin-clients that are able to connect to a load-balanced cluster of GNU/Linux and-or Microsoft Windows terminal servers.

Some of the LTSP-Cluster Features are:

* Central Configuration web interface
* Load balanced thin clients across multiple servers
* Complete autologin support with account creation
* Store hardware information for all clients in the control center

In this tutorial, a basic setup of LTSP-Cluster will be installed. For this purpose, we will use VirtualBox where two x86_64/amd64 Ubuntu servers are configured: the first one will be the root server and the second one the application server. The LTSP-Cluster architecture is presented on the image below and comprises of a root server with a chroot, a load-balancer and a cluster control center; the application server has a LTSBAgent and an account manager running.

INSERT IMAGE OF LTSP-CLUSTER ARCHTECTURE

The network layout built for this tutorial is presented on the following picture. Upfront to this tutorial, the VirtualBox must be configured so that both virtual machines have one network interface connected to NAT and another network inteface host-only.

INSERIR IMAGEM DAS DUAS INTERFACES

After that, create a virtual machine for the thin client. Set 512mb of RAM and configure the system and the network interface like the following images:

INSERIR IMAGEM ABA SISTEMA E ABA NETWORK

Make sure that the adapter type is PCnet-FAST III otherwise the thin client won't be able to receive the image through TFTP.


COLOCAR NO FINAL
After installing the machines, make sure that both servers know each other. For this, you need to edit the `/etc/hosts`: 
```
127.0.0.1       localhost
192.168.1.101   ltsp-root01
192.168.1.102   ltsp-appserv01
```

## Root Server

Install a 64-bit Ubuntu server to install root server. Do not install anything extra – just SSH server. 

Edit the `/etc/network/interfaces` file to set the network interfaces. For the host-only adapter (eth0), set the address as static:

```
auto lo
iface lo inet loopback

auto eth1
iface eth1 inet dhcp

auto eth0
iface eth0 inet static
      address 192.168.1.101
      netmask 255.255.255.0
      gateway 192.168.1.101
```
Reboot the system.
```
sudo reboot
```

The server should have now both interfaces working and access to the internet through NAT. Now,  make all updates and upgrades:

```
sudo apt-get update
sudo apt-get dist-upgrade
```

### Install LTSP-Server and isc-dhcp-server

```
sudo apt-get install ltsp-server isc-dhcp-server
```
The command above will install the ltsp server (which will serve the image to all clients) and the DHCP service.

We must edit the DHCP server configuration file `/etc/dhcp/dhcpd.conf` adding:

```
ddns-update-style none;
default-lease-time 600;
max-lease-time 7200;
authoritative;
log-facility local7;
subnet 192.168.1.0 netmask 255.255.255.0 {
  option domain-name "ubuntu-ltsp5";
  option domain-name-servers 192.168.1.1;
  option routers 192.168.1.1;
  range 192.168.1.200 192.168.1.250;
  next-server 192.168.1.101;
  filename "/ltsp/i386/pxelinux.0";
}
```
Finally, you have to make sure that the file `/etc/default/isc-dhcp-server` has the following line, which will set the isc-dhcp-server to listen on the host-only inteface (eth0) to serve IPs:
```
INTERFACES="eth0"
```

Restart isc-dhcp-server:
```
sudo /etc/init.d/isc-dhcp-server restart
```

If the command above fail, you can see the log on `/var/log/syslog`. You can also test if the isc-dhcp-server is working properly by lauching the thin client virtual machine (you be able to see it getting an IP address in the specified range).

ADICIONAR IMAGEM DE THIN CLIENT RECEBENDO IP

#### Build Chroot

Thin clients need 32-bit chroot. Build that one this way in root server.
```
sudo ltsp-build-client --arch i386 --ltsp-cluster --prompt-rootpass
```

When asked for ltsp-cluster settings answer as follow. Make sure the server name is the IP of the DHCP server for the thin client interface card.
```
Configuration of LTSP-Cluster
NOTE: booleans must be answered as uppercase Y or N
Server name: 192.168.1.101
Port (default: 80): 80
Use SSL [y/N]: N
Enable hardware inventory [Y/n]: Y
Request timeout (default: 2): 2
Root user passwd for chroot will be asked, too.
```
```
Enter new UNIX password: 
Retype new UNIX password: 
passwd: password updated successfully
```

Your answered setup is in this file: /opt/ltsp/i386/etc/ltsp/getltscfg-cluster.conf
```
CC_SERVER=192.168.1.101
PORT=80
ENABLE_SSL=N
INVENTORY=Y
TIMEOUT=2
```
There is a command now that you can use to change into chroot:
```
sudo ltsp-chroot
```

#### Ltsp-cluster-control

Install web based admin program for thin clients in root server.

``` 
sudo apt-get install ltsp-cluster-control postgresql
```

Modify program's configuration file. Note: Do not left any empty lines before or after php-tags (<?php / ?>) - php will not run!
```
sudo nano /etc/ltsp/ltsp-cluster-control.config.php
```
In this setup we use this one. Note all database related information.
```
<?php
    $CONFIG['save'] = "Save";
    $CONFIG['lang'] = "en"; #Language for the interface (en and fr are supported"
    $CONFIG['charset'] = "UTF-8";
    $CONFIG['use_https'] = "false"; #Force https
    $CONFIG['terminal_auth'] = "false";
    $CONFIG['db_server'] = "localhost"; #Hostname of the database server
    $CONFIG['db_user'] = "ltsp"; #Username to access the database
    $CONFIG['db_password'] = "ltsp"; #Password to access the database
    $CONFIG['db_name'] = "ltsp"; #Database name
    $CONFIG['db_type'] = "postgres"; #Database type (only postgres is supported)
    $CONFIG['auth_name'] = "EmptyAuth";
    $CONFIG['loadbalancer'] = "192.168.1.101"; #Hostname of the loadbalancer
    $CONFIG['first_setup_lock'] = "TRUE";
    $CONFIG['printer_servers'] = array("cups.yourdomain.com"); #Hostname(s) of your print servers
    $CONFIG['rootInstall'] = "/usr/share/ltsp-cluster-control/Admin/";
?>
```

Create new user for database. Use same passwd as above (db_password = ltsp)

```
sudo -u postgres createuser -SDRIP ltsp
Enter password for new role: 
Enter it again: 
```

Create new database.
```
sudo -u postgres createdb ltsp -O ltsp
```
Move to the new directory and create tables in database.
```
cd /usr/share/ltsp-cluster-control/DB/
cat schema.sql functions.sql | psql -h localhost ltsp ltsp
Password for user ltsp: 
```
Now you have to act as a root user and move to the /root directory.
```
sudo su
cd /root
```
Get two files for database.
```
wget http://bazaar.launchpad.net/%7Eltsp-cluster-team/ltsp-cluster/ltsp-cluster-control\/download/head%3A/controlcenter.py-20090118065910-j5inpmeqapsuuepd-3/control-center.py
```
```
wget http://bazaar.launchpad.net/%7Eltsp-cluster-team/ltsp-cluster/ltsp-cluster-control\/download/head%3A/rdpldm.config-20090430131602-g0xccqrcx91oxsl0-1/rdp%2Bldm.config
```
Modify control-center.py file, use same information for database as above.
```
nano control-center.py
#/usr/bin/python
import pgdb, os, sys

#FIXME: This should be a configuration file
db_user="ltsp"
db_password="ltsp"
db_host="localhost"
db_database="ltsp"
```
Install one python-package.
```
apt-get install python-pygresql
```
Stop Apache2 and install two files.
```
/etc/init.d/apache2 stop
python control-center.py rdp+ldm.config
```
```
Cleaned status table
Cleaned log table
Cleaned computershw table
Cleaned status table
Cleaned log table
Cleaned computershw table
Regenerated tree
```
Add the following line to the end of `/etc/apache2/apache2.conf` file:
```
Include conf.d/*.conf
```

Start Apache2 again.
```
/etc/init.d/apache2 start
```
Stop acting like a root user.
```
exit
```
Install Xorg and Firefox:
```
sudo aptitude install xorg
sudo apt-get install firefox
```
Open your Firefox and go to the admin web page on `http://ltsp-root01/ltsp-cluster-control/Admin/admin.php`.
```
startx
firefox
```
In the first page (“Configuration”) make a few changes, this way:
```
LANG = en_EN.UTF-8
LDM_DIRECTX = True
LDM_SERVER = %LOADBALANCER%
LOCAL_APPS_MENU = True
SCREEN_07 = ldm
TIMESERVER = ntp.ubuntu.com
XKBLAYOUT = en
``` 
In the tab `Nodes`, create a new node by clicking the button `Create Child` and then typiyng the name of your node (name it ltsp-appserv01).

#### Loadbalancer

Install loadbalancer in root server.
```
sudo apt-get install ltsp-cluster-lbserver
```
Modify information for loadbalancer.
```
sudo nano /etc/ltsp/lbsconfig.xml
```
Here we have only one application server: <node address="http://192.168.1.102:8000" name="ltsp-appserv01"/>

We have changed group name to “karmic” and max-threads to “1”.
```
<?xml version="1.0"?>
<lbsconfig>
    <lbservice listen="*:8008" max-threads="1" refresh-delay="60" returns="$IP"/>
    <lbslave is-slave="false"/>
    <mgmtservice enabled="true" listen="*:8001"/>
    <nodes>
        <group default="true" name="karmic">
            <node address="http://192.168.1.102:8000" name="ltsp-appserv01"/>
        </group>
    </nodes>
    <rules>
        <variable name="LOADAVG" weight="50">
            <rule capacity=".7"/>
        </variable>
        <variable name="NBX11SESS" weight="25">
            <rule capacity="$CPUFREQ*$CPUCOUNT*$CPUCOUNT/120" critical="$CPUFREQ*$CPUCOUNT*$CPUCOUNT/100"/>
        </variable>
        <variable name="MEMUSED" weight="25">
            <rule capacity="$MEMTOTAL-100000"/>
        </variable>
    </rules>
</lbsconfig>
```
The application server ltsp-appserv01 has been set as the deafault application server.

We have now root server ready.

## Application Server

Install a 64-bit Ubuntu server to install root server. Do not install anything extra. 

Edit the `/etc/network/interfaces` file to set the network interfaces. For the host-only adapter (eth0), set the address as static:

```
auto lo
iface lo inet loopback

auto eth1
iface eth1 inet dhcp

auto eth0
iface eth0 inet static
      address 192.168.1.102
      netmask 255.255.255.0
      gateway 192.168.1.102
```
Reboot the system.
```
sudo reboot
```
The server should have now both interfaces working and access to the internet through NAT. Now, make all updates and upgrades:

```
sudo apt-get update
sudo apt-get dist-upgrade
```
At this point, the Application Server and Root Server should be able to communicate. If not, review your network configurations.

Install the following packages:
```
sudo apt-get install ubuntu-desktop ltsp-server ltsp-cluster-lbagent ltsp-cluster-accountmanager
```
Remove following service:
```
sudo update-rc.d -f nbd-server remove
sudo update-rc.d -f gdm remove
sudo update-rc.d -f bluetooth remove
sudo update-rc.d -f pulseaudio remove
```
Create following file and copy this inside that file:
```
sudo nano /etc/xdg/autostart/pulseaudio-module-suspend-on-idle.desktop

[Desktop Entry]
Version=1.0
Encoding=UTF-8
Name=PulseAudio Session Management
Comment=Load module-suspend-on-idle into PulseAudio
Exec=pactl load-module module-suspend-on-idle
Terminal=false
Type=Application
Categories=
GenericName=
```
Create a test user and add user to the following groups:
```
sudo adduser ltsp001
sudo adduser ltsp001 fuse
sudo adduser ltsp001 audio
sudo adduser ltsp001 video
```
You now have a working Application Server.

## Running

To make sure everything works as expected, turn on your Application Server and only after turn on your Root Server. In the root server, the `/var/log/ltsp-cluster-lbserver.log` log file should look like this:

INSERT IMAGE OF LOG

Turn on your Thin Client machine. As this computer is not assigned to a node yet, it will show the following screen upon successful boot:

INSERT INFO SCREEN

To add the thin client computer to a node, open the ltsp-cluster center and ADD HERE HOW TO 









## TROUBLESHOOTING:

You may encouter the following error upon your thin client boot:
```
./screen_session: 48: [: Illegal number:
./screen_session: 78: ./screen_session: /usr/share/ltsp/screen.d/: Permission denied
```
To fix this, substitute the content of the file `/opt/ltsp/amd64/usr/share/ltsp/screen_session` with the content below:
```
#!/bin/sh
#
#  Copyright (c) 2002 McQuillan Systems, LLC
#
#  Author: James A. McQuillan <jam@McQuil.com>
#
#  2005, Matt Zimmerman <mdz@canonical.com>
#  2006, Oliver Grawert <ogra@canonical.com>
#  2007, Scott Balneaves <sbalneav@ltsp.org>
#  2008, Warren Togami <wtogami@redhat.com>
#        Stephane Graber <stgraber@ubuntu.com>
#        Vagrant Cascadian <vagrant@freegeek.org>
#        Gideon Romm <ltsp@symbio-technologies.com>
#  2012, Alkis Georgopoulos <alkisg@gmail.com>
#  2014, Maximiliano Boscovich <maximiliano@boscovich.com.ar>
#  
#  This program is free software; you can redistribute it and/or
#  modify it under the terms of the GNU General Public License as
#  published by the Free Software Foundation; either version 2 of the
#  License, or (at your option) any later version.
#
#  This program is distributed in the hope that it will be useful,
#  but WITHOUT ANY WARRANTY; without even the implied warranty of
#  MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
#  GNU General Public License for more details.
#
#  You should have received a copy of the GNU General Public License
#  along with this program.  If not, you can find it on the World Wide
#  Web at http://www.gnu.org/copyleft/gpl.html, or write to the Free
#  Software Foundation, Inc., 51 Franklin St, Fifth Floor, Boston,
#  MA 02110-1301, USA.
#

# Load LTSP configuration
# ltsp_config sources ltsp-client-functions
. /usr/share/ltsp/ltsp_config

case "$1" in
    [0-1][0-9])
        export SCREEN_NUM="$1"
        ;;
    *)
        die "Usage: $0 [01..12]"
        ;;
esac

while true; do
    # Wait until this is the active vt before launching the screen script
    while [ $(fgconsole) -ne "$SCREEN_NUM" ]; do
        sleep 2
    done

    if [ -f /etc/ltsp/getltscfg-cluster.conf ]; then
        # Reset the environement
        unset $(env | egrep '^(\w+)=(.*)$' | egrep -vw 'PWD|USER|PATH|HOME|SCREEN_NUM' | /usr/bin/cut -d= -f1)
        . /usr/share/ltsp/ltsp_config
        eval $(getltscfg-cluster -a -l prompt)
    fi

    read script args <<EOF
$(eval echo "\$SCREEN_$SCREEN_NUM")
EOF

    # Screen scripts in /etc override those in /usr
    unset script_path
    for dir in /etc/ltsp/screen.d /usr/share/ltsp/screen.d; do
        if [ -x "$dir/$script" ]; then
            script_path="$dir/$script"
            break
        fi
    done
    if [ -z "$script_path" ]; then
        die "Script '$script' for SCREEN_$SCREEN_NUM was not found"
    fi

    for script in $(run_parts_list /usr/share/ltsp/screen-session.d/ S); do
        . "$script"
    done
    "$script_path" "$args"
    for script in $(run_parts_list /usr/share/ltsp/screen-session.d/ K); do
        . "$script"
    done
done
``` 

After that, update the ltsp image:
```
ltsp-update-image i386
```

No network access through NAT: set nat interface as default 
To see which is your default gateway, run:ip route.
To delete the current default gateway, run: sudo route delete default gw
<IP Address> <Adapter>.
To add a new default gateway, run: sudo route add default gw <IP
Address> <Adapter>.
      
      

