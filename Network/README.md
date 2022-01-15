How to Create a Private Network in Proxmox

In this article, we will look at how to create an internal network within a single Proxmox host.

    Patrick Jennings
    Patrick Jennings

    Atlien, computer scientist, car enthusiast, and all around good guy.

    More posts by Patrick Jennings.
    Patrick Jennings

Patrick Jennings
27 Dec 2018  3 min read
How to Create a Private Network in Proxmox
Introduction

Proxmox is a great platform for creating virtual machines and containers. Based on Debian, it uses native KVM and LXC support for virtualization as well as LVM, Ceph, and ZFS for storage. Proxmox also supports out of the box clustering which can be used to easily migrate instances to other connected nodes as well as live snapshots and a great web remote console.

In this article, we will look at how to create an internal network within a single Proxmox host. You will want to follow this guide if you:

    Have a limited number of upstream IP addresses.
    Want an internally NAT-ed subnet optionally with DHCP.

Architecture Overview

We will be architecting the network based on the Proxmox wiki article for Masquerading (NAT) with iptables.

We will be creating a unique Linux bridged interface for the subnet. iptables will be used to route the internal traffic. And dnsmasq will be used to handle DHCP requests.
Proxmox Host

SSH into the Proxmox host and edit /etc/network/interfaces to look like the following:

auto vmbr0
iface vmbr0 inet static
        address  192.168.1.116
        netmask  255.255.255.0
        gateway  192.168.1.1
        bridge-ports eno2
        bridge-stp off
        bridge-fd 0

auto vmbr1
iface vmbr1 inet static
        address  10.10.10.1
        netmask  255.255.255.0
        bridge_ports none
        bridge_stp off
        bridge_fd 0

        post-up echo 1 > /proc/sys/net/ipv4/ip_forward
        post-up   iptables -t nat -A POSTROUTING -s '10.10.10.0/24' -o vmbr0 -j MASQUERADE
        post-down iptables -t nat -D POSTROUTING -s '10.10.10.0/24' -o vmbr0 -j MASQUERADE

vmbr1 is the newly created bridged interface we will use for the private network. When creating VMs or containers, we will be selecting this interface.

Afterwards, execute the following to reload the changes:

/etc/init.d/networking restart

Verification

Check that the new interface is up:

$ ip a
...
26: vmbr1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether fe:77:3e:14:bc:5a brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.1/24 brd 10.10.10.255 scope global vmbr1
       valid_lft forever preferred_lft forever
    inet6 fe80::a82f:ebff:fee4:f45c/64 scope link
       valid_lft forever preferred_lft forever

Check that the static route exists:

$ ip route
...
10.10.10.0/24 dev vmbr1 proto kernel scope link src 10.10.10.1

At this point, you should be able to create a container or VM with a static IPv4 address using the newly created network.
DHCP Server (Optional)

Create a container with the following attributes. I will be utilizing the Ubuntu 18.04 LXC template.

    network: vmbr1
    Static (IPv4): 10.10.10.2/24
    Gateway (IPv4): 10.10.10.1

On the container, install Dnsmasq and disable the systemd resolver if needed.

sudo apt-get install dnsmasq
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved

Create a new file, /etc/dnsmasq.d/vnet, which will be used to define the subnet.

# /etc/dnsmasq.d/vnet
dhcp-range=10.10.10.3,10.10.10.100,12h
dhcp-option=option:dns-server,10.10.10.2

Finally, start Dnsmasq and enable it to start on boot.

sudo systemctl start dnsmasq
sudo systemctl enable dnsmasq

That is all that is needed. Now, the vmbr1 network interface can be selected when creating a VM or container, and DHCP can be used to properly obtain an IP address from the DHCP server.
