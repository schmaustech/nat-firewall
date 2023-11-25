# **Simplicity of Linux Routing Brings OpenShift Portability**

Anyone who has ever done a proof of concept at a customer site knows how daunting it can be.  There is allocating the customers environment from a physical space perspective, power and cooling and then the elephant in the room networking.   Networking always tends to be the most challenging because the way a customer architects and secures their network varies from each and every customer.   Hence when delivering a proof of concept wouldn't it be awesome if all we needed was a single ipaddress and uplink for connectivity?   Linux has always given us the capability to provide such a simple elegant solution.  It's the very reason why router distros like OPNsense, OpenWRT, pfSense and IPFire are based on Linux.  In the following blog I will review configuring such a state with the idea of providing the simplicity of a single uplink for a proof of concept.

In my example I need to deliver a working Red Hat OpenShift 3 node compact cluster to the customers site with a fourth node acting as gateway box which is also running some infrastructure components and a switch to tie it alltogether.  In the diagram below we can see the layout of the configuration and how the networking is setup.   We can see I have a interface one the gateway node that is connected to the upstream network or maybe even the internet depending on circumstances and then another internal interface which is connected to the internal network switch.  All the OpenShift nodes are connected to the internal network switch as well.

~~~bash
[root@bmo ~]# firewall-cmd --get-active-zone
public
  interfaces: enp2s0 enp1s0
[root@bmo ~]# EXTERNAL=enp1s0
[root@bmo ~]# INTERNAL=enp2s0
[root@bmo ~]# firewall-cmd --set-default-zone=internal
success
[root@bmo ~]# firewall-cmd --change-interface=$EXTERNAL --zone=external --permanent
The interface is under control of NetworkManager, setting zone to 'external'.
success
[root@bmo ~]# firewall-cmd --change-interface=$INTERNAL --zone=internal --permanent
The interface is under control of NetworkManager, setting zone to 'internal'.
success
[root@bmo ~]# firewall-cmd --zone=external --add-masquerade --permanent
Warning: ALREADY_ENABLED: masquerade
success
[root@bmo ~]# firewall-cmd --zone=internal --add-masquerade --permanent
success
[root@bmo ~]# firewall-cmd --direct --permanent --add-rule ipv4 nat POSTROUTING 0 -o $EXTERNAL -j MASQUERADE
success
[root@bmo ~]# firewall-cmd --direct --permanent --add-rule ipv4 filter FORWARD 0 -i $INTERNAL -o $EXTERNAL -j ACCEPT
success
[root@bmo ~]# firewall-cmd --direct --permanent --add-rule ipv4 filter FORWARD 0 -i $EXTERNAL -o $INTERNAL -m state --state RELATED,ESTABLISHED -j ACCEPT
success
[root@bmo ~]# firewall-cmd --reload
success
[root@bmo ~]# firewall-cmd --get-active-zone
external
  interfaces: enp1s0
internal
  interfaces: enp2s0
~~~

~~~bash
[root@bmo ~]# firewall-cmd --list-all --zone=external
external (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp1s0
  sources:
  services: ssh
  ports:
  protocols:
  forward: no
  masquerade: yes
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
[root@bmo ~]# firewall-cmd --list-all --zone=internal
internal (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp2s0
  sources:
  services: cockpit dhcpv6-client mdns samba-client ssh
  ports:
  protocols:
  forward: no
  masquerade: yes
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
~~~

~~~bash
[root@bmo ~]# firewall-cmd --permanent --zone=external --add-service=https
success
[root@bmo ~]# firewall-cmd --permanent --zone=internal --add-service=https
success
[root@bmo ~]# firewall-cmd --permanent --zone=external --add-forward-port=port=443:proto=tcp:toport=443:toaddr=192.168.100.72
success
[root@bmo ~]# firewall-cmd --reload
success
[root@bmo ~]# firewall-cmd --list-all --zone=external
external (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp1s0
  sources:
  services: https ssh
  ports:
  protocols:
  forward: no
  masquerade: yes
  forward-ports:
        port=443:proto=tcp:toport=443:toaddr=192.168.100.72
  source-ports:
  icmp-blocks:
  rich rules:
[root@bmo ~]# firewall-cmd --list-all --zone=internal
internal (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp2s0
  sources:
  services: cockpit dhcpv6-client https mdns samba-client ssh
  ports:
  protocols:
  forward: no
  masquerade: yes
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
~~~


~~~bash
[root@router ~]# firewall-cmd --permanent --new-policy policy_int_to_ext
success
[root@router ~]# firewall-cmd --permanent --policy policy_int_to_ext --add-ingress-zone internal
success
[root@router ~]# firewall-cmd --permanent --policy policy_int_to_ext --add-egress-zone external
success
[root@router ~]# firewall-cmd --permanent --policy policy_int_to_ext --set-priority 100
success
[root@router ~]# firewall-cmd --permanent --policy policy_int_to_ext --set-target ACCEPT
success
[root@router ~]# firewall-cmd --reload
success
[root@router ~]# firewall-cmd --permanent --zone=internal --add-service=dns
success
[root@router ~]# firewall-cmd --reload~~~


~~~bash
firewall-cmd --permanent --new-policy policy_int_to_ext
firewall-cmd --permanent --policy policy_int_to_ext --add-ingress-zone internal
firewall-cmd --permanent --policy policy_int_to_ext --add-egress-zone external
firewall-cmd --permanent --policy policy_int_to_ext --set-priority 100
firewall-cmd --permanent --policy policy_int_to_ext --set-target ACCEPT
firewall-cmd --reload
~~~

~~~bash
# curl https://192.168.0.70 -k
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title></title>
  <base href="/">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="icon" type="image/x-icon" href="favicon.ico">
<link rel="stylesheet" href="styles.df04819ee43f8b8ff5a1.css"></head>
<body>
  <app-root></app-root>
<script src="runtime-es2015.66c79b9d36e7169e27b0.js" type="module"></script><script src="runtime-es5.66c79b9d36e7169e27b0.js" nomodule defer></script><script src="polyfills-es5.6e1055b712d0b250b19e.js" nomodule defer></script><script src="polyfills-es2015.6022d6f28e0500e60d30.js" type="module"></script><script src="scripts.9ae077a2cc1f84a7f419.js" defer></script><script src="main-es2015.d2879de224f20b99fee3.js" type="module"></script><script src="main-es5.d2879de224f20b99fee3.js" nomodule defer></script></body>
</html>
~~~
