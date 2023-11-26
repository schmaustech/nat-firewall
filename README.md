# **Simplicity of Linux Routing Brings OpenShift Portability**

<img src="lab.jpg" style="width: 1000px;" border=0/>

Anyone who has ever done a proof of concept at a customer site knows how daunting it can be.  There is allocating the customers environment from a physical space perspective, power and cooling and then the elephant in the room networking.   Networking always tends to be the most challenging because the way a customer architects and secures their network varies from each and every customer.   Hence when delivering a proof of concept wouldn't it be awesome if all we needed was a single ipaddress and uplink for connectivity?   Linux has always given us the capability to provide such a simple elegant solution.  It's the very reason why router distros like OPNsense, OpenWRT, pfSense and IPFire are based on Linux.  In the following blog I will review configuring such a state with the idea of providing the simplicity of a single uplink for a proof of concept.

In this example I wanted to deliver a working Red Hat OpenShift 3 node compact cluster that I could bring anywhere.   A fourth node acting as the gateway box will also run some infrastructure components with a switch to tie it all together.  In the diagram below we can see the layout of the configuration and how the networking is setup.  I should note this could use four physical boxes or in my testing I had all 4 nodes virtualized.   We can see I have a interface enp1s0 on the gateway node that is connected to the upstream network or maybe even the internet depending on circumstances and then another internal interface enp2s0 which is connected to the internal network switch.  All the OpenShift nodes are connected to the internal network switch as well.  The internal network will never change but the external network could be anything and could change if we wanted it to.   What this means when bringing this setup to another location is I just need to update the enp1s0 interface with the right ipaddress, gateway and nameserver.   Further to ensure the OpenShift api and ingress wildcards resolve the external DNS (whatever controls that) will need those two records added and pointed to the enp1s0 interface.   Nothing changes on the OpenShift cluster nodes or gateway node configuration of dhcp or bind.

<img src="poc-in-box.png" style="width: 1000px;" border=0/>

The gateway node has Red Hat Enterprise Linux 9.3 installed on it along with dhcp and bind services both of which are listening only on the internal enp2s0 interface.  In order to have the proper network address translation and service redirection we need to modify the default firewalld configuration on the box.  

First let's go ahead and see what the active zone is with firewalld.   We will find that both interfaces are in the public zone.

~~~bash
# firewall-cmd --get-active-zone
public
  interfaces: enp2s0 enp1s0
~~~

We will first set our two interfaces to variables to make the rest of the commands easy to follow.  Interface enp1s0 will be set to external and enp2s0 will be set to internal.  Then we will go ahead and create an internal zone.  Note we do not need to create an external zone because one exists by default with firewalld.   We can then assign the interfaces to their proper zones.

~~~bash
# EXTERNAL=enp1s0
# INTERNAL=enp2s0

# firewall-cmd --set-default-zone=internal
success

# firewall-cmd --change-interface=$EXTERNAL --zone=external --permanent
The interface is under control of NetworkManager, setting zone to 'external'.
success

# firewall-cmd --change-interface=$INTERNAL --zone=internal --permanent
The interface is under control of NetworkManager, setting zone to 'internal'.
success
~~~

Next we can enable masquerading between the zones.  We will find that by default masquerading was enabled by default for the external zone.  However if one chose different zone names we need to point out that both need to be set.

~~~bash
# firewall-cmd --zone=external --add-masquerade --permanent
Warning: ALREADY_ENABLED: masquerade
success

# firewall-cmd --zone=internal --add-masquerade --permanent
success
~~~

Now we can add the rules to forward traffic between zones.

~~~bash
# firewall-cmd --direct --permanent --add-rule ipv4 nat POSTROUTING 0 -o $EXTERNAL -j MASQUERADE
success

# firewall-cmd --direct --permanent --add-rule ipv4 filter FORWARD 0 -i $INTERNAL -o $EXTERNAL -j ACCEPT
success

# firewall-cmd --direct --permanent --add-rule ipv4 filter FORWARD 0 -i $EXTERNAL -o $INTERNAL -m state --state RELATED,ESTABLISHED -j ACCEPT
success
~~~

At this point let's go ahead and reload our firewall and show the active zones again.  Now we should see our interfaces are in their proper zones and active. 

~~~bash
# firewall-cmd --reload
success

# firewall-cmd --get-active-zone
external
  interfaces: enp1s0
internal
  interfaces: enp2s0
~~~

If we look at each zone we can see the configuration that currently exists for each zone.

~~~bash
# firewall-cmd --list-all --zone=external
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

# firewall-cmd --list-all --zone=internal
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

The zones need to be updated so we can ensure any external traffic bound for https and port 6443 is sent to the OpenShift ingress virtual ipaddress and OpenShift api virual ipaddress respectively.   We also need to allow for DNS resolution traffic internally on the internal zone.

~~~bash
# firewall-cmd --permanent --zone=external --add-service=https
success
# firewall-cmd --permanent --zone=internal --add-service=https
success
# firewall-cmd --permanent --zone=external --add-forward-port=port=443:proto=tcp:toport=443:toaddr=192.168.100.72
success
# firewall-cmd --reload
success
# firewall-cmd --list-all --zone=external
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

Up to this point we would have a working setup if we were on Red Hat Enterprise Linux 8.x.  However there were changes made with Red Hat Enterprise Linux 9.x and hence we need to add a internal to external policy to ensure traffic flow.

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
