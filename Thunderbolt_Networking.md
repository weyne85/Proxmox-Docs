# Thunderbolt Networking

[this gist is part of this series](/76e94832927a89d977ea989da157e9dc)

#### NOTE FOR THIS TO BE RELIABLE ON NODE RESTARTS YOU WILL NEED PROXMOX KERNEL 6.2.16-14-pve  OR HIGER
This fixes issues i bugged with the thunderbolt / thunderbolt-net maintainers (i will take everyones thanks now, lol) 

## Install LLDP - this is great to see what nodes can see which.
-  install lldpctl with `apt install lldpd`

## Load Kernel Modules
- add `thunderbolt` and `thunderbolt-net` kernel modules (this must be done all nodes - yes i know it can sometimes work withoutm but the thuderbolt-net one has interesting behaviou' so do as i say - add both ;-)
    1. `nano /etc/modules` add modules at bottom of file, one on each line	
    2. save using `x` then `y` then `enter`

## Prepare /etc/network/interfaces
doing this means we don't have to give each thunderbolt a manual IPv6 addrees and that these addresses stay constant no matter what
Add the following to each node using `nano /etc/network/interfaces`

If you see any sections called thunderbolt0 or thunderbol1 delete them at this point.

Now add the following (note we will set IP addresses in the UI):

```
allow-hotplug en05
iface en05 inet manual
       mtu 65520

iface en05 inet6 manual
        mtu 65520

allow-hotplug en06
iface en06 inet manual
        mtu 65520

iface en06 inet6 manual
        mtu 65520

```

If you see any thunderbol sections delete them from the file before you save it.

## Rename Thunderbolt Connections
This is needed as proxmox doesn't recognize the thunderbolt interface name.  There are various methods to do this. This method was selected after trial and error because:
- the thunderboltX naming is not fixed to a port (it seems to be based on sequence you plug the cables in)
- the MAC address of the interfaces changes with most cable insertion and removale events


1. use `udevadm monitor` command to find your device IDs when you insert and remove each TB4 cable. Yes you can use other ways to do this, i recommend this one as it is great way to understand what udev does - the command proved more useful to me than `the syslog` or `lspci command` for troublehsooting thunderbolt issues and behavious.  In my case my two pci paths are `0000:00:0d.2`and `0000:00:0d.3` if you bought the same hardware this will be the same on all 3 units. Don't assume your PCI device paths will be the same as mine.

3. create a link file using `nano /etc/systemd/network/00-thunderbolt0.link` and enter the following content:

```
[Match]
Path=pci-0000:00:0d.2
Driver=thunderbolt-net
[Link]
MACAddressPolicy=none
Name=en05
```
3. create a second link file using `nano /etc/systemd/network/00-thunderbolt1.link` and enter the following content:
```
[Match]
Path=pci-0000:00:0d.3
Driver=thunderbolt-net
[Link]
MACAddressPolicy=none
Name=en06
```

## Set Interfaces to UP on reboots and cable insertions
This section en sure that the interfaces will be brought up at boot or cable insertion with whatever settings are in /etc/network/interfaces  - this shouldn't need to be done, it seems like a bug in the way thunderbolt networking is handled (i assume this is debian wide but haven't checked).

1. create a udev rule to detect for cable insertion using `nano /etc/udev/rules.d/10-tb-en.rules` with the following content:
```
ACTION=="move", SUBSYSTEM=="net", KERNEL=="en05", RUN+="/usr/local/bin/pve-en05.sh"
ACTION=="move", SUBSYSTEM=="net", KERNEL=="en06", RUN+="/usr/local/bin/pve-en06.sh"
```
2. save the file

3. create the first script referenced above using `nano /usr/local/bin/pve-en05.sh` and with the follwing content:
```
#!/bin/bash

# this brings the renamed interface up and reprocesses any settings in /etc/network/interfaces for the renamed interface
/usr/sbin/ifup en05
```
save the file and then

3. create the second script referenced above using `nano /usr/local/bin/pve-en06.sh` and with the follwing content:
```
#!/bin/bash

# this brings the renamed interface up and reprocesses any settings in /etc/network/interfaces for the renamed interface
/usr/sbin/ifup en06
```
and save the file

4. make both scripts executable with `chmod +x /usr/local/bin/*.sh`
5. Reboot (restarting networking, init 1 and init 3 are not good enough, so reboot)

## Enabling IP Connectivity
[proceed to the next gist](/4c664734535da122f4ab2951b22b2085)
