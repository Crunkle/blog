---
layout: post
title: Leave the old iptables for firewalld
---

I still encounter people using iptables as their primary interface for their firewall. This isn't necessarily a bad thing, but you can make your life easier (and sometimes more secure) by using a firewall daemon. I am going to be focusing on a quick runthrough of firewalld; a user-friendly way to configure iptables.

You first need to install firewalld on your system and enable it at boot time. I am using CentOS 7 on my system so the package manager and service manager may differ depending on your distribution.

```shell
[root@alpha ~]# yum -y install firewalld
Package firewalld-0.3.9-14.el7.noarch already installed and latest version
[root@alpha ~]# systemctl enable firewalld
```

If all has been done correctly, you should now see that listing all rules using iptables will look rather confusing. This is normal as we are going to be moving towards the firewalld command set and no longer need to use iptables as the frontend. Just to make sure everything is running as expected, list all firewall rules currently implemented.

```shell
[root@alpha ~]# firewall-cmd --list-all
public (default, active)
  interfaces: eth0
  sources:
  services: ssh
  ports:
  masquerade: no
  forward-ports:
  icmp-blocks:
  rich rules:
```

Don’t get confused by the look of this as it is actually very simple. The first thing people worry about is the zone at the top (public). This does not mean that all ports are open and your server is insecure! In iptables you add rules to chains in order to configure your firewall. This is slightly different as it uses zones, services and ports. Ports are simply just port numbers which are accepted through your firewall, services are named ports (for example, http is the equivalent to TCP port 80) and zones are different groups of rules. By default, your firewall will be in the public zone which is just a name for a group. Besides some ICMP changes between groups, with the exception to the drop and block groups, most groups behave exactly the same and you should be able to pick your group based on its name.

```shell
[root@alpha ~]# firewall-cmd --get-active-zones
dmz
  interfaces: eth0
```

You can quickly see which zones are attached to which interfaces. I personally use dmz as it is tuned towards publicly accessible networks with limited access to the internal network. In order to change your zone, you need to attach an interface to a zone or change your default zone.

```shell
[root@alpha ~]# firewall-cmd --set-default-zone=dmz
success
[root@alpha ~]# firewall-cmd --permanent --add-interface=eth0
success
[root@alpha ~]# firewall-cmd --reload
success
[root@alpha ~]# firewall-cmd --list-all
dmz (default, active)
  interfaces: eth0
  sources:
  services: ssh
  ports:
  masquerade: no
  forward-ports:
  icmp-blocks:
  rich rules:
```

You should now see that you have moved your interfaces to use a specific zone (or group) and from now on, all traffic passing through your server will use the rules defined in the group you specified. Many people notice that we are reloading before listing rules; this is only for permanent rules. All rules in your group can either be a temporary rule or a permanent rule. You will usually want to set permanent rules as these persist after reboot and reload. If you do not specify the permanent flag, all of your rules will disappear. On the downside, by using the permanent flag you need to reload your firewall to see the changes. You are now ready to open some ports to your server.

```shell
[root@alpha ~]# firewall-cmd --permanent --add-port=1234/udp
success
[root@alpha ~]# firewall-cmd --permanent --add-port=80/tcp
success
[root@alpha ~]# firewall-cmd --reload
success
[root@alpha ~]# firewall-cmd --list-all
dmz (default, active)
  interfaces: eth0
  sources:
  services: ssh
  ports: 80/tcp 1234/udp
  masquerade: no
  forward-ports:
  icmp-blocks:
  rich rules:
```

That was easy! You have now opened UDP port 1234 and TCP port 80 (also known as http). If you recall from earlier, we can replace common ports (such as port 80) with their service name for ease of use. You can also define your own services if they aren’t already available (but that is out of the scope of this runthrough).

```shell
[root@alpha ~]# firewall-cmd --permanent --add-service=ftp
success
[root@alpha ~]# firewall-cmd --reload
success
[root@alpha ~]# firewall-cmd --list-all
dmz (default, active)
  interfaces: eth0
  sources:
  services: ftp ssh
  ports: 80/tcp 1234/udp
  masquerade: no
  forward-ports:
  icmp-blocks:
  rich rules:
```

You do not need both the port and the service added. Once you add the service, you can remove the port you originally added and simply use the service name to manage the firewall as a replacement. Other features are available in firewalld, such as rich rules, but almost everything can be done with simple groups. For example, you may only want specific IP addresses able to access your SSH console as an extra layer of security. To do this, instead of using rich rules or going back to iptables, you can add these IP addresses to a work group and only allow the work group access to SSH. Be sure to play around with the interface as, once you learn the syntax, you will never want to go back.
