# Enable Dual Stack (IPv4 and IPv6) OpenFabric Routing

This will result in an IPv4 and IPv6 routable mesh network that can survive any one node failure or any one cable failure. Alls the steps in this section must be performed on each node

Note for ceph do not dual stack - either use IPv4 or IPv6 addressees for all the monitors, MDS and daemons - despite the docs implying it is ok my findings on quincy are is it is funky.... 

[this gist is part of this series](/76e94832927a89d977ea989da157e9dc)

## Create Loopback interfaces
Doing this means we don't have to give each thunderbolt a manual IPv6 or IPv4 addrees and that these addresses stay constant no matter what.
Add the following to each node using `nano /etc/network/interfaces`

This should go uder the `auto lo` section and for each node the X should be 1, 2 or depending on the node 

```
auto lo:0
iface lo:0 inet static
        address 10.0.0.8X/32
        
auto lo:6
iface lo:6 inet static
        address fc00::8X/128
```

so on the first node it would look comething like this:

```
...
auto lo
iface lo inet loopback
 
auto lo:0
iface lo:0 inet static
        address 10.0.0.81/32

auto lo:6
iface lo:6 inet static
        address fc00::81/128
...
```

also add this is as the last line to the interfaces file

```
# This must be the last line in the file
post-up /usr/bin/systemctl restart frr.service
```

Save file, repeat on each node.

## Enable IPv4 and IPv6 forwarding
1. use `nano /etc/sysctl.conf` to open the file
2. uncomment `#net.ipv6.conf.all.forwarding=1` (remove the # symbol)
3. uncomment `#net.ipv4.ip_forward=1` (remove the # symbol)
4. save the file
5. reboot?

## FRR Setup

### Install FRR
Install Free Range Routing (FRR) `apt install frr`

### Enable the fabricd daemon

1. edit the frr daemons file (`nano /etc/frr/daemons`) to change `fabricd=no` to `fabricd=yes` 
2. save the file
3. restart the service with `systemctl restart frr`


### Configure OpenFabric (perforn on all nodes)
1. enter the FRR shell with `vtysh`
2. optionally show the current config with `show running-config`
3. enter the configure mode with `configure`
4. Apply the bellow configuration (it is possible to cut and paste this into the shell instead of typing it manually, you may need to press return to set the last !.  Also check there were no errors in repsonse to the paste text.).

**Note: the X should be the number of the node you are working on, as an example -  0.0.0.1, 0.0.0.2 or 0.0.0.3.**
```
ip forwarding
ipv6 forwarding
!
frr version 8.5.2
frr defaults traditional
hostname pve1
service integrated-vtysh-config
!
interface en05
ip router openfabric 1
ipv6 router openfabric 1
exit
!
interface en06
ip router openfabric 1
ipv6 router openfabric 1
exit
!
interface lo
ip router openfabric 1
ipv6 router openfabric 1
openfabric passive
exit
!
router openfabric 1
net 49.0000.0000.000X.00
exit
!

```
5. you may need to pres return after the last `!` to get to a new line - if so do this
6. exit the configure mode with the command `end`
7. save the configu with `write memory`
8. show the configure applied correctly with `show running-config` - note the order of the items will be different to how you entered them and thats ok.  (If you made a mistake i found the easiest way was to edt `/etc/frr/frr.conf` - but be careful if you do that.)
9. use the command `exit` to leave setup
10. repeat steps 1 to 9 on the other 3 nodes

11. once you have configured all 3 nodes issue the command `vtysh -c "show openfabric topology"` if you did everything right you will see:
```
Area 1:
IS-IS paths to level-2 routers that speak IP
Vertex               Type         Metric Next-Hop             Interface Parent
pve1                                                                  
10.0.0.81/32         IP internal  0                                     pve1(4)
pve2                 TE-IS        10     pve2                 en06      pve1(4)
pve3                 TE-IS        10     pve3                 en05      pve1(4)
10.0.0.82/32         IP TE        20     pve2                 en06      pve2(4)
10.0.0.83/32         IP TE        20     pve3                 en05      pve3(4)

IS-IS paths to level-2 routers that speak IPv6
Vertex               Type         Metric Next-Hop             Interface Parent
pve1                                                                  
fc00::81/128         IP6 internal 0                                     pve1(4)
pve2                 TE-IS        10     pve2                 en06      pve1(4)
pve3                 TE-IS        10     pve3                 en05      pve1(4)
fc00::82/128         IP6 internal 20     pve2                 en06      pve2(4)
fc00::83/128         IP6 internal 20     pve3                 en05      pve3(4)

IS-IS paths to level-2 routers with hop-by-hop metric
Vertex               Type         Metric Next-Hop             Interface Parent

```
Now you should be in a place to ping each node from evey node across the thunderbolt mesh using IPv4 or IPv6 as you see fit.
