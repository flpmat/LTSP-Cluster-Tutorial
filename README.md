# LTSP-Cluster (Tutorial)

LTSP-Cluster is a set of LTSP plugins and client-side tools that allows you to deploy and centrally manage large numbers of thin-clients. It allows you to run thousands of thin-clients that are able to connect to a load-balanced cluster of GNU/Linux and-or Microsoft Windows terminal servers.

Some of the LTSP-Cluster Features are:

* Central Configuration web interface
* Load balanced thin clients across multiple servers
* Complete autologin support with account creation
* Store hardware information for all clients in the control center

In this tutorial, a basic setup of LTSP-Cluster will be installed. For this purpose, two x86_64/amd64 servers are configured: the first one is a root server and the second one is the application server. The LTSP-Cluster architecture is presented on the image below and comprises of a root server with a chroot, a load-balancer and a cluster control center; the application server has a LTSBAgent and an account manager running.

The network layout built for this tutorial is presented on the following picture. Upfront to this tutorial, the VirtualBox must be configured so as both virtual machines have one network interface connected to NAT and one network inteface bridged.

INSERIR IMAGEM DE CONFIGURAÇÃO DE REDE

Both servers must know each other. For this, you need to edit the `/etc/hosts`: 
```
127.0.0.1       localhost
192.168.1.101   ltsp-root01
192.168.1.102   ltsp-appserv01
```

## Root Server

Install a 64-bit Ubuntu server to install root server. Do not install anything extra – just SSH server. Make all updates and upgrades:

```
sudo apt-get update
sudo apt-get dist-upgrade
```
Edit the `/etc/network/interfaces` file to set the networkd interfaces. For the Bridge card, set the address as static:

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
Reset the network services so the changes take place.
```
ifconfig eth0 down & ifconfig eth0 up && ifconfig eth1 down & ifconfig eth1 up

OR

sudo /etc/init.d/networking restart
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
Finally, you have to make sure that the file `/etc/default/isc-dhcp-serve` has the following line, which will set the isc-dhcp-server to listen on the bridged inteface (eth1) to serve IPs:
```
INTERFACES="eth1"
```




This will install the ltsp server that will be used to serve the image to all clients and the DHCP server that has changed name, package name and executable name without being documented ANYWHERE. If you are going to do something do it well or don't do it at all, thats what I always say but lets continue.

Now we need to edit the configuration file for the DHCP server so lets open that file
?
1
sudo pico /etc/dhcp/dhcpd.conf

The idea here is that since you are looking to create a cluster with a lot of thin clients, you will need to have a large pool of IP addresses. I am giving you an example here with a supernetted Class C network that gives us over 1000 usable IPs. A standard Class C or /24 subnet will work as well.
