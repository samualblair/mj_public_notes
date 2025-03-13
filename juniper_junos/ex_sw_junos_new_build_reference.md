# EX Switch Junos new build reference
--- 

## Basic starting point likey via console
Commands to start CLI , as console (with root login) will default to csh not junos 'cli' shell.
```
cli
```

## Commands to edit config
```
config
```

## Stop system from trying to do auto provisioning, otherwise console gets spammed
```
delete chassis auto-image-upgrade
```

## For Cisco compatability - send STP messages on all vlans not just vlan 1
## set protocols vstp vlan all

## For root ssh access if you want it allowed - otherwise setup new admin user
```
set system services ssh root-login allow
```

## Set Hostname
```
set system host-name SW-ACCESS-1
```

## Set the System Root Password via Input Text
```
set system root-authentication plain-text-password
```
password will need to then be entered


## Other system wide settings to consider:
```
set forwarding-options storm-control-profiles default all
set protocols lldp interface all
set protocols lldp-med interface all
set protocols igmp-snooping vlan default
set protocols rstp interface all
set protocols vstp vlan all
```


# Management port configurations

Some platforms have 1x 'me' interface that can be configured, some have 2x 'me'

If you confiure under 'vme' then it works regardless

If you configure under 'me0' or 'me1' then of course the config is locked to that given interface

More fun on larger chassis with blades where even 'me0' may float between blades/supervisors/routing-engines

## Disable DHCP , and set Static IP on dedicated mangement port
```
deactivate interfaces vme unit 0 family inet dhcp
set interfaces vme unit 0 family inet address 10.10.0.29/24
```

## IRB (Integrated Routing-Bridging)
Also Known as - SVI (Switch Virtual Interface)

Alternativly or in addition setup on irb.0 interface  -- aka SVI for vlan 1
```
deactivate interfaces irb unit 0 family inet dhcp
set interfaces irb unit 0 family inet address 10.10.0.29/24
```

## Alternativly or in addition setup on irb.5 interface -- aka irb.x SVI for vlan xxx - example VLAN 5
```
set interfaces irb unit 5 family inet address 10.10.0.29/24
```

## Set up a system default route - assuming DHCP didn't pick up address and provide route
```
set routing-options static route 0.0.0.0/0 next-hop 10.10.0.1
set system name-server 8.8.8.8 
set system name-server 1.1.1.1
```

# Vlan and Data Uplink configurations

## Set Some VLANs
```
set vlans Wired-Users-1 vlan-id 301
set vlans Net-Mgmt-1 vlan-id 5
```

## Configs shown below are more manual denfintions
These can be done in 'template' and then applied to groups of ports if desired

For copper uplink - or fiber but slower aka 1g
```
set interfaces ge-0/0/0 native-vlan-id 1
set interfaces ge-0/0/0 unit 0 family ethernet-switching vlan members Wired-Users-1
set interfaces ge-0/0/0 unit 0 family ethernet-switching vlan members Net-Mgmt-1
set interfaces ge-0/0/0 unit 0 family ethernet-switching interface-mode trunk
```

For 10g Uplink - speed is what changes it to 'xe' , not physical port placement
```
set interfaces xe-0/0/0 native-vlan-id 1
set interfaces xe-0/0/0 unit 0 family ethernet-switching vlan members Wired-Users-1
set interfaces xe-0/0/0 unit 0 family ethernet-switching vlan members Net-Mgmt-1
set interfaces xe-0/0/0 unit 0 family ethernet-switching interface-mode trunk
```

## Set port for management connection
If using in-band management VLAN instead of (or in addition to) dedicated mangement port
```
set interfaces ge-0/0/1 unit 0 family ethernet-switching vlan members Net-Mgmt-1
```

# Configuring LACP / LAG / Port-Channel

## Set all the relevant settings for Agg , and define new 'aeX' interface (will be newly created)
```
set interfaces ae0 description Uplink-Agg-to-Core
set interfaces ae0 vlan-tagging
set interfaces ae0 native-vlan-id 1
```

## Includes agg settings unique to an agg
```
set interfaces ae0 aggregated-ether-options link-speed 10g
set interfaces ae0 aggregated-ether-options lacp active
set interfaces ae0 aggregated-ether-options lacp periodic fast
```

## Includes settings that would normally be on a single port but are instead on the Agg
```
set interfaces ae0 unit 0 family ethernet-switching interface-mode trunk
set interfaces ae0 unit 0 family ethernet-switching vlan members Wired-Users-1
set interfaces ae0 unit 0 family ethernet-switching vlan members Net-Mgmt-1
set interfaces ae0 unit 0 family ethernet-switching storm-control default
```

## And simply activate port to be part of lag, other than amybe speed settings port should have no real config on it
## Port 1 example - notice it calls out ae0
```
set interfaces xe-0/2/0 description Uplink-Agg-to-Core-Member-Port
set interfaces xe-0/2/0 gigether-options 802.3ad ae0
```

## Port 2 example - notice it calls out ae0
```
set interfaces xe-1/2/0 description Uplink-Agg-to-Core-Member-Port
set interfaces xe-1/2/0 gigether-options 802.3ad ae0
```

# Vlan and Data Uplink configurations - Large groups , using templates

## Can set something like STP to run as RSTP not normal STP for all interfaces easily here
```
set protocols rstp interface all
```

## Very common to want to turn on 'Edge' ports and 'BPDU guard'
Also you may want to forward this over to Sysmex, it's for edge port configurations if they want to use them

## Aka BPDU Guard
```
set protocols rstp bpdu-block-on-edge
```
## Set Edge port (aka disable stp, so better have bpdu guard on)
This example is for all ports in the 'user_access_ports' group

```
set protocols rstp interface user_access_ports edge
```

## You can also override (disable bpdu-guard) on one of those ports if needed directly, for example:
```
set ethernet-switching-options bpdu-block interface ge-4/0/1 disable
```

# For other common port configs, profiles can be used
For example groups such as a 'server port' or 'ap port' or 'user port' , these can be built as profiles first

## Group 1 example 'server_trunk_ports'
Just about anyting that can be consistent across them can be defined here

Things like native vlan
```
set interface interface-range server_trunk_ports native-vlan-id 100
```

Things like vlans on a trunk
```
set interface interface-range server_trunk_ports unit 0 family ethernet-switching interface-mode trunk
set interface interface-range server_trunk_ports unit 0 family ethernet-switching vlan members Legacy-Data-1
set interface interface-range server_trunk_ports unit 0 family ethernet-switching vlan members WEBDMZ
```

Things like storm control
```
set interface interface-range server_trunk_ports unit 0 family ethernet-switching storm-control default
```

## Group 2 example 'ap_trunk_ports'
```
set interface interface-range ap_trunk_ports native-vlan-id 10
set interface interface-range ap_trunk_ports unit 0 family ethernet-switching interface-mode trunk
set interface interface-range ap_trunk_ports unit 0 family ethernet-switching vlan members AP-Mgmt-1
set interface interface-range ap_trunk_ports unit 0 family ethernet-switching vlan members Guest-Wifi-1
set interface interface-range ap_trunk_ports unit 0 family ethernet-switching vlan members Wireless-Users-1
set interface interface-range ap_trunk_ports unit 0 family ethernet-switching storm-control default
```

## Group 3 example 'net_mgmt_access_ports'
```
set interface interface-range net_mgmt_access_ports unit 0 family ethernet-switching vlan members Net-Mgmt-1
set interface interface-range net_mgmt_access_ports unit 0 family ethernet-switching storm-control default
```

## Then you just define which port(s) are part of the group(s)
```
set interface interface-range net_mgmt_access_ports member ge-0/0/2
set interface interface-range server_trunk_ports member ge-0/0/3
set interface interface-range server_trunk_ports member ge-0/0/4
set interface interface-range server_trunk_ports member ge-0/0/5
set interface interface-range ap_trunk_ports member mge-0/0/45
set interface interface-range ap_trunk_ports member mge-0/0/46
set interface interface-range ap_trunk_ports member mge-0/0/47
```

# Config cleanup - 'remove' config that makes no sense
You want to clean up / remove config that is not used or relevent to prevent future confusion or issues.

For example you may have a bunch of ge-0/0/x present, and mge-0/0/x , or xe-0/0/x , te-0/0/x present...

But you KNOW that 'ge' config would be use if running 1g, and 'mge' would be used when running 2.5+

Or you KNOW that the SFP is going to be 10G so 'xe-0/0/0' and not 100g 'te-0/0/0'

## You can remove extra config by simply deleting it out of the config file (can always add it back later if needed later)
```
delete interface ge-0/0/22
delete interface ge-0/0/23
delete interface mge-0/0/24
delete interface xe-0/0/25
delete interface te-0/0/26
```

# Save / Activate (commit) - Check

## Verify pending changes
```
show | compare
```

## Compare Changes / Rollbacks
Default is 'compare rollback 0' and 0 isn't needed , can also type older compare such as 'compare rollback 4'

```
show | compare rollback
```

Rollback to older config (doesn't apply until commit)

```
rollback 1
```

## Optionally check before commit
The system would be done anyway when commiting, but can be done while testing some new config and still working

```
commit check
```

## Commands to commit
Perform commit (check is done by system if not valid won’t commit, activates and ‘saves’ config). Will keep user in config cli after commit is done.
```
commit
```

Shortand to commit and then quit config mode (go back to run cli).
```
commit and-quit
```

Commit config, but don't save it, and auto-rollback in 10 minutes if a normal 'commit' is not done.
```
commit confirmed 10
```

Then exit/quit to logout fully of config mode (assuming user didn't use 'and-quit'). Perform a second time to log out of device.
```
exit
```
