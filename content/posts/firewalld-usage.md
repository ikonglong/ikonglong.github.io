---
title: "Firewalld Usage"
slug: ""
date: 2022-07-16T16:48:01+08:00
lastmod: 2022-07-16T16:48:01+08:00
draft: false 
toc: true
weight: false
categories: ["操作系统"]
tags: ["linux", "firewall"]
---

This post shows the common ways in which the `firewalld` command is used.
<!--more-->

## Introduction to firewalld

The dynamic firewall daemon **firewalld** provides a **dynamically managed firewall** with support for network “zones” to assign a level of trust to a network and its associated connections and interfaces. It has support for IPv4 and IPv6 firewall settings. It supports Ethernet bridges and has a separation of runtime and permanent configuration options. It also has an interface for services or applications to add firewall rules directly.

## Understanding firewalld

**firewall-config** is a graphical configuration tool that is used to configure firewalld.
The **firewall service** provided by firewalld is dynamic rather than static because changes to the configuration can be made at anytime and are immediately implemented, there is no need to save or apply the changes. No unintended disruption of existing network connections occurs as no part of the firewall has to be reloaded.
**firewall-cmd** is a command line client for firewalld.

## Comparison of firewalld to system-config-firewall and iptables

The essential differences between firewalld and the iptables service are:
The iptables service stores configuration in /etc/sysconfig/iptables while firewalld stores it in various XML files in /usr/lib/firewalld/ and /etc/firewalld/. Note that the /etc/sysconfig/iptables file does not exist as firewalld is installed by default on Red Hat Enterprise Linux.

With the iptables service, every single change means flushing all the old rules and reading all the new rules from /etc/sysconfig/iptables while with firewalld there is no re-creating of all the rules; only the differences are applied. Consequently, firewalld can change the settings during runtime without existing connections being lost.
Both use iptables tool to talk to the kernel packet filter.

![The Firewall Stack](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Security_Guide/images/firewall_stack.png "The Firewall Stack")

## Understanding Network Zones

Firewalls can be used to separate networks into different zones based on the level of trust the user has decided to place on the devices and traffic within that network. NetworkManager informs firewalld to which zone an interface belongs. An interface's assigned zone can be changed by NetworkManager or via the firewall-config tool which can open the relevant NetworkManager window for you.

The zone settings in /etc/firewalld/ are a range of preset settings which can be quickly applied to a network interface. They are listed here with a brief explanation:

**drop**  
Any incoming network packets are dropped, there is no reply. Only outgoing network connections are possible.

**block**  
Any incoming network connections are rejected with an icmp-host-prohibited message for IPv4 and icmp6-adm-prohibited for IPv6. Only network connections initiated from within the system are possible.

**public**  
For use in public areas. You do not trust the other computers on the network to not harm your computer. Only selected incoming connections are accepted.

**external**  
For use on external networks with masquerading enabled especially for routers. You do not trust the other computers on the network to not harm your computer. Only selected incoming connections are accepted.

**dmz**  
For computers in your demilitarized zone that are publicly-accessible with limited access to your internal network. Only selected incoming connections are accepted. *关于 DMZ，更好的解释请看 [鸟哥的私房菜-防火墙的一般网络布线示意](http://linux.vbird.org/linux_server/0250simple_firewall.php#firewall_topo)*

**work**  
For use in work areas. You mostly trust the other computers on networks to not harm your computer. Only selected incoming connections are accepted.

**home**  
For use in home areas. You mostly trust the other computers on networks to not harm your computer. Only selected incoming connections are accepted.

**internal**  
For use on internal networks. You mostly trust the other computers on the networks to not harm your computer. Only selected incoming connections are accepted.

**trusted**  
All network connections are accepted.

It is possible to designate one of these zones to be the default zone. When interface connections are added to NetworkManager, they are assigned to the default zone. **On installation, the default zone in firewalld is set to be the public zone.**

## Choosing a Network Zone

The network zone names have been chosen to be self-explanatory and to allow users to quickly make a reasonable decision. However, **a review of the default configuration settings should be made and unnecessary services disabled according to your needs and risk assessments.**

## Understanding Predefined Services

A service can be a list of local ports and destinations as well as a list of firewall helper modules automatically loaded if a service is enabled. The use of predefined services makes it easier for the user to enable and disable access to a service.
The services are specified by means of individual XML configuration files which are named in the following format: *service-name.xml*.

**List the default predefined services available:**

```bash
> ls /usr/lib/firewalld/services/
```

{{< admonition type=note title="注意" open=true >}}
Files in `/usr/lib/firewalld/services/` must not be edited. Only the files in `/etc/firewalld/services/` should be edited.
{{< /admonition >}}

**List the system or user created services:**

```bash
> ls /etc/firewalld/services/
```

**Add or change a service:**

```bash
# Use files in /usr/lib/firewalld/services/ as templates
> cp /usr/lib/firewalld/services/[service].xml /etc/firewalld/services/[service].xml
```

firewalld will prefer files in /etc/firewalld/services/ but will fall back to /usr/lib/firewalld/services/ should a file be deleted, but only after a reload.

## Understanding the Direct Interface

firewalld has a so called “direct interface”, which enables directly passing rules to iptables, ip6tables and ebtables. It is intended for use by applications and not users. The direct interface mode is intended for services or applications to add specific firewall rules during runtime.

```bash
> firewall-cmd --direct ...
```

## Using the iptables Service

Use the iptables and ip6tables services instead of firewalld.

```bash
# Disable firewalld
> systemctl disable firewalld
> systemctl stop firewalld

# Package iptables-services contains iptables and ip6tables service
> yum install iptables-services
> systemctl start iptables
> systemctl start ip6tables
> systemctl enable iptables
> systemctl enable ip6tables
```

## Configuring Firewall Using firewall-cmd

Check firewall-cmd version:

```bash
> firewall-cmd --version
```

View help:

```bash
> firewall-cmd --help
```

为了让所做的设置永久生效，需要给 除添加有 --direct 选项的命令之外的 所有命令添加 **--permanent**。若无，则设置仅在下一次 **firewall-cmd --reload**、system boot、firewalld service restart之前有效。**reloading firewall本身并不会破坏已有网络连接，但请注意这样做会丢弃已做出的短暂的设置修改。**

## View the Firewall Settings Using CLI

Check the state of firewalld:

```bash
> firewall-cmd --state
```

View the list of active zones, with a list of the interfaces currently assigned to them:

```bash
> firewall-cmd --get-active-zones
public
  interfaces: em1
```

Find out the zone that an interface, for example em1, is currently assigned to:

```bash
> firewall-cmd --get-zone-of-interface=em1
public
```

Find out all the interfaces assigned to a zone:

```bash
> firewall-cmd --zone=public --list-interfaces
em1 wlan0
```

This information is obtained from **NetworkManager** and only shows interfaces, not connections.

Find out all the settings of a zone:

```bash
> firewall-cmd --zone=public --list-all
public
  interfaces:
  services: mdns dhcpv6-client ssh
  ports:
  forward-ports:
  icmp-blocks: source-quench
```

View the list of services currently loaded:

```bash
> firewall-cmd --get-services
cluster-suite pop3s bacula-client smtp ipp radius bacula ftp mdns samba dhcpv6-client dns openvpn imaps samba-client http https ntp vnc-server telnet libvirt ssh ipsec ipp-client amanda-client tftp-client nfs tftp libvirt-tls
```

This includes all loaded **predefined services** and **custom services**.

List all custom services, even not yet loaded:

```bash
> firewall-cmd --permanent --get-services
```

## Change the Firewall Settings Using CLI

### Drop All Packets (Panic Mode 恐慌模式)

Start dropping all incoming and outgoing packets:

```bash
> firewall-cmd --panic-on
```

Active connections will be terminated after a period of inactivity; the time taken depends on the individual session time out values.

Disable panic mode:

```bash
> firewall-cmd --panic-off
```

After disabling panic mode, established connections might work again if panic mode was enabled for a short period of time.

Find out if panic mode is enabled or disabled:

```bash
> firewall-cmd --query-panic
```

Prints yes with exit status 0, if enabled, prints no with exit status 1 otherwise.

### Reload the Firewall Using CLI

Reload the firewall with out interrupting user connections, that is to say, with out losing state info:

```bash
> firewall-cmd --reload
```

Reload the firewall and interrupt user connections, that is to say, to discard state info:

```bash
> firewall-cmd --complete-reload
```

This command should normally only be used in case of severe firewall problems. For example, if there are state info problems and no connection can be established but the firewall rules are correct.

### Add an Interface to a Zone

Add an Interface to a Zone Using CLI:

```bash
> firewall-cmd --zone=public --add-interface=em1
```

To make this setting permanent, add the **--permanent** option and reload the firewall.

Add an Interface to a Zone by Editing the Interface Configuration File (e.g. add em1 to the work zone):

> ZONE=work

Note that if you omit the ZONE option, or use ZONE=, or ZONE='', then the default zone will be used. **NetworkManager** will automatically reconnect and the zone will be set accordingly.

### Get/Set the Default Zone

Print default zone for connections and interfaces:

```bash
> firewall-cmd --get-default-zone
```

Set the Default Zone by Using CLI(Command Line Interface):

```bash
> firewall-cmd --set-default-zone=public
```

This change will take immediate effect and in this case it is not necessary to reload the firewall.

Set the Default Zone by Editing the firewalld Configuration File:

Edit /etc/firewalld/firewalld.conf as follows:

> \# default zone
> \# The default zone used if an empty zone string is used.
> \# Default: public
> DefaultZone=home

Reload the firewall

```bash
> firewall-cmd --reload
```

This will reload the firewall without losing state information (TCP sessions will not be interrupted).

### Query/Open/Close Ports in the Firewall Using CLI

List all open ports for a zone:

```bash
> firewall-cmd --zone=dmz --list-ports
```

Note that this will not show ports opened as a result of the --add-services command.

Query ports to check if they're open:

```bash
> firewall-cmd --zone=public --query-port=5060-5061/udp
yes
```

Close ports:

```bash
> firewall-cmd --permanent --zone=public --remove-port=5060-5061/udp
success
```

Add a port to a zone (e.g. allow TCP traffic to port 8080 to the dmz zone):

```bash
> firewall-cmd --zone=dmz --add-port=8080/tcp
```

To make this setting permanent, add the --permanent option and reload the firewall.

Add a range of ports to a zone (e.g. allow the ports from 5060 to 5061 to the public zone):

```bash
> firewall-cmd --zone=public --add-port=5060-5061/udp
```

### Add a Service to a Zone

Add a service to a zone (e.g. allow SMTP to the work zone):

```bash
> firewall-cmd --zone=work --add-service=smtp
```

Add a Service to a Zone by Editing XML Files, see [How to using firewalls](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Security_Guide/sec-Using_Firewalls.html)

### Remove a Service from a Zone

Remove a service from a zone(e.g. remove SMTP from the work zone):

```bash
> firewall-cmd --zone=work --remove-service=smtp
```

Remove a Service from a Zone by Editing XML files, see [How to using firewalls](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Security_Guide/sec-Using_Firewalls.html).

### Configure IP Address Masquerading

Check if IP masquerading is enabled for the given zone:

```bash
> firewall-cmd --zone=external --query-masquerade
```

Prints `yes` with exit status 0, if enabled, prints `no` with exit status 1 otherwise. If zone is omitted, the default zone will be used.

Enable IP masquerading:

```bash
> firewall-cmd --zone=external --add-masquerade
```

Disable IP masquerading:

```bash
> firewall-cmd --zone=external --remove-masquerade
```

### Configure Port Forwarding Using CLI

To forward inbound network packets from one port to an alternative port or address, **first** enable IP address masquerading for a zone.

```bash
> firewall-cmd --zone=external --add-masquerade
```

To forward packets to a local port, that is to say to a port on the same system

```bash
> firewall-cmd --zone=external --add-forward-port=port=22:proto=tcp:toport=3753
```

In this example, the packets intended for port 22 are now forwarded to port 3753.
*--add-forward-port=port=22* 这部分貌似有问题，应该删掉一个"port="，待求证！
To forward packets to another IPv4 address, usually an internal address, without changing the destination port.

```bash
> firewall-cmd --zone=external --add-forward-port=port=22:proto=tcp:toaddr=192.0.2.55
```

In this example, the packets intended for port 22 are now forwarded to the same port at the address given with the toaddr.
To forward packets to another port at another IPv4 address, usually an internal address.

```bash
> firewall-cmd --zone=external /
      --add-forward-port=port=22:proto=tcp:toport=2055:toaddr=192.0.2.55
```

In this example, the packets intended for port 22 are now forwarded to port 2055 at the address given with the toaddr option.  

## Resources

- [How to using firewalls](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Security_Guide/sec-Using_Firewalls.html)
- [fedora firewalld wiki](https://fedoraproject.org/wiki/FirewallD?rd=FirewallD/)
- [鸟哥的私房菜-防火墙的一般网络布线示意](http://linux.vbird.org/linux_server/0250simple_firewall.php#firewall_topo)
