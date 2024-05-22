## Enable OSPF Routing on Thunderbolt Mesh 
# This has been deprectaed
It is now superceded by **Openfabric Routing** [see here](/4c664734535da122f4ab2951b22b2085) 

### continue at your own peril, for reference only now.

## Old Gist
This will result in an IPv4 routable mesh network that can survive any one node failure or any one cable failure.
All the steps in this section *must be performed on each node*

Please note the main section of this gist describes IPv4 on mesh. Lower down you will find additonal files that cpatures:
1. differences if you want dual stack IPv4 / IPv6 routing (this is now what i run since writing the original gist)
2. opernfabric instead of OSPF (to do)

[this gist is part of this series](/76e94832927a89d977ea989da157e9dc)

## Key Parameters
Key Information Used
Note i used the 10.x IPv4 space as this is not used anywhere else on my network YMMV

lo = loopback
en05/06 - these are the thunderbolt ports


**Node l:**
- lo:0 = 10.0.0.81/32
- en05 = 10.0.0.5/30
- en06 = 10.0.0.9/30
- ospf router-id = 0.0.0.1

**Node 2:**
- lo:0 = 10.0.0.82/32
- en05 = 10.0.0.10/30
- en06 = 10.0.0.13/30
- ospf router-id = 0.0.0.2

**Node 3:**
- lo:0 = 10.0.0.83/32
- en05 = 10.0.0.14/30
- en06 = 10.0.0.6/30
- ospf router-id = 0.0.0.3

## Enable IPv4 forwarding
Using IPv4 to take advantage of not needing to use addresses - does make things simpler 
-  uncomment `#net.ipv4.ip_forward=1` using `nano /etc/sysctl.conf` (remove the # symbol and save the file)

## Create Loopback interface
doing this means we don't have to give each thunderbolt a manual IPv6 addrees and that these addresses stay constant no matter what
Add the following to each node using `nano /etc/network/interfaces`

This should go uder the `auto lo` section and for each node the X should be 1, 2 or depending on the node 

```
auto lo:0
iface lo:0 inet static
        address 10.0.0.8X/32
```

so on the first node it would look comething like this:

```
...
auto lo
iface lo inet loopback
 
auto lo:0
iface lo:0 inet static
        address 10.0.0.81/32
...
```

Save file.

## Assign IP address to en05 and en06 using the GUI
1. use the table further up and assign addresses
2. after appliying both addresss remeber to hit `apply configuration` button


## Install OSPF (perform on all nodes)
1. Install Free Range Routing (FRR) `apt install frr`
2. Edit the FRR config file: `nano /etc/frr/daemons`
3. Adjust `ospfd=no` to `ospfd=yes`
4. save the file
5. restart the service with `systemctl restart frr`

### Configure OSPF (perforn on all nodes)
1. enter the FRR shell with `vtysh`
2. optionally show the current config with `show running-config`
3. enter the configure mode with `configure`
4. Apply the bellow configuration (it is possible to cut and paste this into the shell instead of typing it manually, you may need to press return to set the last !.  Also check there were no errors in repsonse to the paste text.).
Note: the X should be the number of the node you are working on, so for my stetup this would 0.0.0.1, 0.0.0.2 or 0.0.0.3.
```
ip forwarding
!
router ospf
 ospf router-id 0.0.0.X
 log-adjacency-changes
 exit
!
interface lo
 ip ospf area 0
 exit
!
interface en05
 ip ospf area 0
 ip ospf network broadcast
 exit
!
interface en06
 ip ospf area 0
 ip ospf network broadcast
 exit
!

```
5. you may need to pres return after the last `!` to get to a new line - if so do this
6. exit the configure mode with the command `end`
7. save the configu with `write memory`
8. show the configure applied correctly with `show running-config` - note the order of the items will be different to how you entered them and thats ok.  (If you made a mistake i found the easiest way was to edt `/etc/frr/frr.conf` - but be careful if you do that.)
9. use the command `exit` to leave setup
10. repeat steps 1 to 9 on the other 3 nodes
11. once you have configured all 3 nodes issue the command `vtysh -c "show ip ospf neighbor"` you will see:
```
root@pve1:~# vtysh -c "show ip ospf neighbor"

Neighbor ID     Pri State           Up Time         Dead Time Address         Interface                        RXmtL RqstL DBsmL
0.0.0.2           1 Full/DROther    52m26s            33.951s 10.0.0.10       en06:10.0.0.9                        0     0     0
0.0.0.3           1 Full/DROther    51m56s            33.444s 10.0.0.6        en05:10.0.0.5                        0     0     0

```
10. now issue the command `vtysh -c "show ip route"` and you will see:
```
root@pve1:~# vtysh -c "show ip route"
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

C>* 10.0.0.4/30 is directly connected, en05, 00:53:16
O>* 10.0.0.5/32 [110/0] is directly connected, en05, weight 1, 00:53:16
O   10.0.0.6/32 [110/10] via 10.0.0.6, en05 inactive, weight 1, 00:53:11
C>* 10.0.0.8/30 is directly connected, en06, 00:53:46
O>* 10.0.0.9/32 [110/0] is directly connected, en06, weight 1, 00:53:46
O   10.0.0.10/32 [110/10] via 10.0.0.10, en06 inactive, weight 1, 00:53:41
O>* 10.0.0.13/32 [110/10] via 10.0.0.10, en06, weight 1, 00:53:32
O>* 10.0.0.14/32 [110/10] via 10.0.0.6, en05, weight 1, 00:53:11
O   10.0.0.81/32 [110/0] is directly connected, lo, weight 1, 12:15:09
C>* 10.0.0.81/32 is directly connected, lo, 12:15:09
O>* 10.0.0.82/32 [110/10] via 10.0.0.10, en06, weight 1, 00:53:41
O>* 10.0.0.83/32 [110/10] via 10.0.0.6, en05, weight 1, 00:53:11
C>* 192.168.1.0/24 is directly connected, vmbr0, 12:15:06
```
and lastly `ip route`
```
root@pve1:~# ip route
default via 192.168.1.1 dev vmbr0 proto kernel onlink 
10.0.0.4/30 dev en05 proto kernel scope link src 10.0.0.5 
10.0.0.8/30 dev en06 proto kernel scope link src 10.0.0.9 
10.0.0.12/30 nhid 53 proto ospf metric 20 
        nexthop via 10.0.0.6 dev en05 weight 1 
        nexthop via 10.0.0.10 dev en06 weight 1 
10.0.0.82 nhid 54 via 10.0.0.10 dev en06 proto ospf metric 20 
10.0.0.83 nhid 33 via 10.0.0.6 dev en05 proto ospf metric 20 
192.168.1.0/24 dev vmbr0 proto kernel scope link src 192.168.1.81 
```

##Testing Example
You can now test the network by pinging the IPv4 loopback addresses of the other nodes.  For example ping (using my IPs defined earlier):
- `ping 10.0.0.81`
- `ping 10.0.0.82`
- `ping 10.0.0.83`

Now pull one of the TB cables and repeat the test.

You should still be able to ping all nodes!!
