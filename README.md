# LTSP-Cluster (Tutorial)

LTSP-Cluster is a set of LTSP plugins and client-side tools that allows you to deploy and centrally manage large numbers of thin-clients. It allows you to run thousands of thin-clients that are able to connect to a load-balanced cluster of GNU/Linux and-or Microsoft Windows terminal servers.

Some of the LTSP-Cluster Features are:

* Central Configuration web interface
* Load balanced thin clients across multiple servers
* Complete autologin support with account creation
* Store hardware information for all clients in the control center

In this tutorial, a basic setup of LTSP-Cluster will be installed. For this purpose, we will use VirtualBox where two x86_64/amd64 Ubuntu servers are configured: the first one will be the root server and the second one  the application server. The LTSP-Cluster architecture is presented on the image below and comprises of a root server with a chroot, a load-balancer and a cluster control center; the application server has a LTSBAgent and an account manager running.

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
