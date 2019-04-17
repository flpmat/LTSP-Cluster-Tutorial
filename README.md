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

### Install LTSP-Server and isc-dhcp-server

```
sudo apt-get install ltsp-server isc-dhcp-server
```

