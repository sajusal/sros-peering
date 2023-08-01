# Nokia SROS Peering Configuration

This page provides the basic step-by-step configuration required to set up a Nokia 7750 Service Router as a Peering router. All the required feature sets for a peering router are covered here with configuration and show examples. Most sections also provide links to Nokia documentation for further reading.

All configurations are in MD-CLI flat format. Reference chassis is 7750 SR-1 and software version is SROS 23.7R1. Use `show system info` command to verify your router's chassis model and software version.

Disclaimer: This is not an exhaustive list of all the features and associated options on SROS for peering. For more details on the features and options, please refer to the documentation links in each section.

# Topology

We will be using the topology in the [Containerlab peering lab](https://containerlab.dev/lab-examples/peering-lab/).

In this topology, the 7750 SR router is connected to a FRR peer and 2 Route servers - OpenBGPd and BIRD.

![image](/topology.png)

# Switching to MD-CLI

Starting SROS release 23, all new routers will default to MD-CLI on boot up. In case you are using an older release and need to switch to MD-CLI, please use the below commands which will enable MD-CLI, Netconf and gNMI on the router. For more details on MD-CLI, visit [SROS MD-CLI Guide](https://documentation.nokia.com/sr/23-7-1/titles/md-cli-command-reference.html#undefined).


From Classic CLI run:

```
configure system management-interface configuration-mode model-driven
logout
```

Login again with your username and password. The router will log you into MD-CLI.

Run the below commands in MD-CLI for your username:

```
configure private
/configure system { management-interface cli md-cli auto-config-save true }
/configure system { management-interface netconf admin-state enable }
/configure system { management-interface netconf auto-config-save true }
/configure system { security user-params local-user user “user1" access netconf true }
/configure system { grpc admin-state enable }
/configure system { grpc allow-unsecure-connection }
/configure system { grpc gnmi admin-state enable }
/configure system { grpc gnmi auto-config-save true }
/configure system { security user-params local-user user “user1" access grpc true }
commit
```

# User and Profile Management

SROS supports local, TACACS, Radius or LDAP for Authentication, Authorization and Accounting (AAA). The below example shows local management.

For more details on AAA, visit [SROS AAA Documentation](https://documentation.nokia.com/sr/23-7-1/books/system-management/security-system-management.html#ai9exj5x89).


Configuring user profile:
```
/configure system security aaa local-profiles profile "NOC-User" default-action deny-all
/configure system security aaa local-profiles profile "NOC-User" entry 10 { match "configure system security" }
/configure system security aaa local-profiles profile "NOC-User" entry 10 { action deny }
/configure system security aaa local-profiles profile "NOC-User" entry 20 { match "show" }
/configure system security aaa local-profiles profile "NOC-User" entry 20 { action permit }
```

Configuring user and associating a profile:
```
/configure system security user-params local-user user "markp" password “Changeme1&"
/configure system security user-params local-user user "markp" access { console true }
/configure system security user-params local-user user "markp" console { member ["NOC-User"] }
```

# Hardware Configuration

Assuming this is a brand new router, the cards should be configured before we proceed with the peering configuration. If this is already done, you can skip this section.

The card and mda types depend on the variant of the 7750 SR in use. The equipped card and mda types can be seen using the `show card state` command In this example, we are using a 7750 SR-1 with 1 MDA.

```
 /configure card 1 card-type iom-1
 /configure card 1 { level he }
 /configure card 1 mda 1 { mda-type me6-100gb-qsfp28 }
```

The state of the card and MDA can be viewed using the below show command:

```
A:admin@SR1# show card state

===============================================================================
Card State
===============================================================================
Slot/  Provisioned Type                  Admin Operational   Num   Num Comments
Id         Equipped Type (if different)  State State         Ports MDA
-------------------------------------------------------------------------------
1      iom-1:he                          up    up                  2
1/1    me6-100gb-qsfp28                  up    up            6
A      cpm-1                             up    up                      Active
===============================================================================
```

# Ports and Interfaces

Physical port is configured first following by an interface with an IPv4 or IPv6 address.

Each port is considered as a ‘connector’ and supports breakout. The breakout type used on the connector should be configured first which then unlocks the individual ports in the breakout for configuration.

Port MTU is 9212 by default.

In this example, we are configuring the connector to use a 1x100G breakout.

```
 /configure port 1/1/c1 { admin-state enable }
 /configure port 1/1/c1 { connector breakout c1-100g }
 commit
 /configure port 1/1/c1/1 { admin-state enable }
 /configure port 1/1/c1/1 { description "To Peering LAN" }
 commit
```

The status of the port can be viewed using the below command:

```
A:admin@SR1# /show port

===============================================================================
Ports on Slot 1
===============================================================================
Port          Admin Link Port    Cfg  Oper LAG/ Port Port Port   C/QS/S/XFP/
Id            State      State   MTU  MTU  Bndl Mode Encp Type   MDIMDX
-------------------------------------------------------------------------------
1/1/c1        Up         Link Up                          conn   100GBASE-LR4*
1/1/c1/1      Up    Yes  Up      9212 9212    - netw null cgige
```

The interface is given a name, IP and associated to a physical port.

```
  /configure router "Base" interface "To-Peering-LAN" port 1/1/c1/1
  /configure router "Base" interface "To-Peering-LAN" ipv4 { primary address 192.168.0.1 prefix-length 24 }
  /configure router "Base" interface "To-Peering-LAN" ipv6 { address 2001:a8::4 prefix-length 124 }
```

The `system` interface is the router's loopback interface (like lo0 or loopback0). The name of this interface cannot be changed. If no explicit `router-id` is configured, the `system` interface IPv4 address is used as the router-id. The `system` interface should be assigned a /32 IP.

```
  /configure router "Base" interface "system" ipv4 { primary address 10.0.0.1 prefix-length 32 }
  /configure router "Base" interface "system" ipv6 { address 2001:1::101 prefix-length 128 }
```

The status of the interfaces can be seen using the below command:

```
A:admin@SR1# show router interface

===============================================================================
Interface Table (Router: Base)
===============================================================================
Interface-Name                   Adm       Opr(v4/v6)  Mode    Port/SapId
   IP-Address                                                  PfxState
-------------------------------------------------------------------------------
To-Peering-LAN                   Up        Up/Up       Network 1/1/c1/1
   192.168.0.1/24                                              n/a
   2001:a8::4/124                                              PREFERRED
   fe80::1668:ffff:fe00:0/64                                   PREFERRED
system                           Up        Up/Up       Network system
   10.0.0.1/32                                                 n/a
   2001:1::101/128                                             PREFERRED
-------------------------------------------------------------------------------
Interfaces : 2
===============================================================================

```

# IGP - OSPF

OSPF or IS-IS will be required to connect with other routers within the same AS. In this example, we are configuring a OSPFv2 neighbor. Port and interface configuration is similar to what is shown in previous section.

For more details on OSPF configuration, visit [SROS OSPF Documentation](https://documentation.nokia.com/sr/23-7-1/books/unicast-routing-protocols/ospf-unicast-routing-protocols.html).

```
    /configure router "Base" ospf 0 admin-state enable
    /configure router "Base" ospf 0 area 0.0.0.0 { interface "Interface-to-R1" interface-type point-to-point }
```

OSPF neighbor status can be seen using the below command:

```
A:admin@SR1# show router ospf neighbor

===============================================================================
Rtr Base OSPFv2 Instance 0 Neighbors
===============================================================================
Interface-Name                   Rtr Id          State      Pri  RetxQ   TTL
   Area-Id
-------------------------------------------------------------------------------
Interface-to-R1                  10.10.20.103    Full       1    0       35
   0.0.0.0
-------------------------------------------------------------------------------
No. of Neighbors: 1
===============================================================================
```

# IGP - IS-IS

IS-IS or OSPF will be required to connect with other routers within the same AS. In this example, we are configuring the router to be in IS-IS Level 2 and also enabled IPv6 native routing (the other option is MT). Port and interface configuration is similar to what is shown in previous section.

For more details on IS-IS configuration, visit [SROS IS-IS Documentation](https://documentation.nokia.com/sr/23-7-1/books/unicast-routing-protocols/is-is-unicast-routing-protocols.html).

```
    /configure router "Base" isis 0 admin-state enable
    /configure router "Base" isis 0 ipv6-routing native
    /configure router "Base" isis 0 level-capability 2
    /configure router "Base" isis 0 system-id 0100.0000.0001
    /configure router "Base" isis 0 area-address [49.0000]
    /configure router "Base" isis 0 interface "Interface-to-R1" { interface-type point-to-point }
```

IS-IS adjacency status can be seen using the below command:

```
A:admin@SR1 show router isis adjacency

===============================================================================
Rtr Base ISIS Instance 0 Adjacency
===============================================================================
System ID                Usage State Hold Interface                     MT-ID
-------------------------------------------------------------------------------
sr103                    L2    Up    25   Interface-to-R1               0
-------------------------------------------------------------------------------
Adjacencies : 1
===============================================================================
```

# BGP

In this example, we are peering with the OpenBGPd and BIRD route servers. Local AS on the SROS is 64501. Remote AS is 64503.

For more details on BGP configuration, visit [SROS BGP Documentation](https://documentation.nokia.com/sr/23-7-1/books/unicast-routing-protocols/bgp-unicast-routing-protocols.html).

```
    /configure router "Base" autonomous-system 64501
    /configure router "Base" bgp router-id 10.0.0.1
    /configure router "Base" bgp group "eBGP-Peering" { type external }
    /configure router "Base" bgp group "eBGP-Peering" { peer-as 64503 }
    /configure router "Base" bgp group "eBGP-Peering" { family ipv4 true }
    /configure router "Base" bgp group "eBGP-Peering" { family ipv6 true }
    /configure router "Base" bgp neighbor "192.168.0.3" { group "eBGP-Peering" }
    /configure router "Base" bgp neighbor "192.168.0.4" { group "eBGP-Peering" }
```

BGP neighbor status can be seen using the below command.

```
A:admin@SR1# show router bgp summary
===============================================================================
 BGP Router ID:10.0.0.1         AS:64501       Local AS:64501
===============================================================================
BGP Admin State         : Up          BGP Oper State              : Up
Total Peer Groups       : 1           Total Peers                 : 2
Total VPN Peer Groups   : 0           Total VPN Peers             : 0
Current Internal Groups : 1           Max Internal Groups         : 1
Total BGP Paths         : 21          Total Path Memory           : 7416

-- snip --

===============================================================================
BGP Summary
===============================================================================
Legend : D - Dynamic Neighbor
===============================================================================
Neighbor
Description
                   AS PktRcvd InQ  Up/Down   State|Rcv/Act/Sent (Addr Family)
                      PktSent OutQ
-------------------------------------------------------------------------------
192.168.0.3
                64503      20    0 00h07m26s 3/0/0 (IPv4)
                           19    0           0/0/0 (IPv6)
192.168.0.4
                64503      27    0 00h10m05s 2/0/0 (IPv4)
                           41    0           1/0/0 (IPv6)
-------------------------------------------------------------------------------
```

To list the received routes from a neighbor, use the below command:

```
A:admin@SR1# show router bgp neighbor 192.168.0.3 received-routes
===============================================================================
 BGP Router ID:10.0.0.1         AS:64501       Local AS:64501
===============================================================================
 Legend -
 Status codes  : u - used, s - suppressed, h - history, d - decayed, * - valid
                 l - leaked, x - stale, > - best, b - backup, p - purge
 Origin codes  : i - IGP, e - EGP, ? - incomplete

===============================================================================
BGP IPv4 Routes
===============================================================================
Flag  Network                                            LocalPref   MED
      Nexthop (Router)                                   Path-Id     IGP Cost
      As-Path                                                        Label
-------------------------------------------------------------------------------
i     10.10.1.24/29                                      n/a         None
      192.168.0.3                                        None        0
      64503                                                          -
i     10.10.20.103/32                                    n/a         None
      192.168.0.3                                        None        0
      64503                                                          -
i     192.168.0.0/24                                     n/a         None
      192.168.0.3                                        None        0
      64503                                                          -
-------------------------------------------------------------------------------
Routes : 3
===============================================================================
```

To monitor the number of routes installed per line card, use the below command:

```
A:admin@SR1# show router fib 1 summary ipv4

===============================================================================
FIB Summary
===============================================================================
                              Active
-------------------------------------------------------------------------------
Static                        0
Direct                        3
Host                          0
BGP                           0
BGP VPN                       0
--snip---
VPN Leak                      0
-------------------------------------------------------------------------------
Total Installed               4
-------------------------------------------------------------------------------
Current Occupancy             1%
Overflow Count                0
Suppressed by Selective FIB   0
Occupancy Threshold Alerts
    Alert Raised 0 Times;
===============================================================================
```

# Route Policies

Routing policies control the size and content of the routing tables, the routes that are advertised, and the best route to take to reach a destination.

For more details on route policy configuration and options, visit [SROS Route Policies Documentation](https://documentation.nokia.com/sr/23-7-1/books/unicast-routing-protocols/route-policies.html).

In these examples, we are creating AS path lists, community and prefix lists.

```
 /configure policy-options as-path "PEERING" { expression "64503" }
 /configure policy-options as-path-group "BOGON" { entry 10 expression ".* 0 .*" }
 /configure policy-options as-path-group "BOGON" { entry 20 expression ".* [64496-64511] .*" }
 /configure policy-options as-path-group "BOGON" { entry 30 expression ".* 65535 .*" }

 /configure policy-options community "LARGE-PEER" { member "65100:100" }
 /configure policy-options community "SMALL-PEERS" { member "65200:200" }
 /configure policy-options community "SMALL-PEERS" { member "65400:.*$" }
 /configure policy-options community "SMALL-PEERS" { member "65500:.*" }

 /configure policy-options prefix-list "AS65xx-prefixes" { prefix 10.100.100.0/24 type longer }
 /configure policy-options prefix-list "AS65xx-prefixes" { prefix 10.200.0.0/16 type through through-length 24 }
 /configure policy-options prefix-list "AS65xx-prefixes" { prefix 192.168.10.0/24 type through through-length 24 }
 /configure policy-options prefix-list "AS65xx-prefixes" { prefix 10.10.1.1/32 type exact }
 /configure policy-options prefix-list "AS65xx-prefixes" { prefix 172.16.0.0/16 type range start-length 16 }
 /configure policy-options prefix-list "AS65xx-prefixes" { prefix 172.16.0.0/16 type range end-length 19 }
 /configure policy-options prefix-list "IPv6-list" { prefix 2001:fd00:84::/46 type longer }
 /configure policy-options prefix-list "SMALLER_THAN_/48" { prefix ::/0 type range start-length 49 }
 /configure policy-options prefix-list "SMALLER_THAN_/48" { prefix ::/0 type range end-length 128 }
```

Policy statements can be created as below. Entries can be either numbered or named.

```
    /configure policy-options policy-statement "EXT-AS-IMPORT" entry-type named
    /configure policy-options policy-statement "EXT-AS-IMPORT" named-entry "Routes-AS64503" { from as-path name "PEERING" }
    /configure policy-options policy-statement "EXT-AS-IMPORT" named-entry "Routes-AS64503" { action action-type accept }
```

The policy can be applied as import or export under the BGP root, group or neighbor context.

```
/configure router "Base" bgp group "eBGP-Peering" import { policy ["EXT-AS-IMPORT"] }
```

# Test Route Policies

Route policies can be tested and evaluated before they are applied to BGP.

For more details and examples, visit [SROS MD-CLI Command Tree](https://documentation.nokia.com/aces/sr/23-7-1/cli-books/clear-monitor-show-tools-commands/cmst-p-commands.html#ai9j784l5l)

```
A:admin@SR1# show router bgp policy-test plcy-or-long-expr "EXT-AS-IMPORT" family ipv4 prefix 0.0.0.0/0 longer neighbor 192.168.0.3
===============================================================================
 BGP Router ID:10.0.0.1         AS:64501       Local AS:64501
===============================================================================
 Legend -
 Status codes  : u - used, s - suppressed, h - history, d - decayed, * - valid
                 l - leaked, x - stale, > - best, b - backup, p - purge
 Origin codes  : i - IGP, e - EGP, ? - incomplete

===============================================================================
BGP IPv4 Routes
===============================================================================
      Network                                            LocalPref   MED
      Nexthop                                            Path-Id     Label
      As-Path
-------------------------------------------------------------------------------
Accepted by Policy EXT-AS-IMPORT Entry Routes-AS64503
      10.10.1.24/29                                      None        None
      192.168.0.3                                        None        n/a
      64503                                                          -
Accepted by Policy EXT-AS-IMPORT Entry Routes-AS64503
      10.10.20.103/32                                    None        None
      192.168.0.3                                        None        n/a
      64503                                                          -
Accepted by Policy EXT-AS-IMPORT Entry Routes-AS64503
      192.168.0.0/24                                     None        None
      192.168.0.3                                        None        n/a
      64503                                                          -
-------------------------------------------------------------------------------
Routes : 3
===============================================================================
```

# CPM Filter

CPM filters are hardware-based filters used to restrict traffic from the line cards directed to the CPM CPU, such as control and management packets. Separate configuration is required for IPv4 and IPv6 packet matching conditions. Prefix lists can be used for groups of IP addresses.

In this example, we have 3 entries. The 3rd entry will log and accept all unmatched packets after the first 2 entries.

In order to avoid getting locked out of the router while configuring CPM filters, it is recommended to use `commit confirm` when making changes.

For more details on CPM filters, visit [SROS CPM Filter Documentation](https://documentation.nokia.com/sr/23-7-1/books/system-management/security-system-management.html#ai9exj5yai). Also, check out the [SROS Security Guide](https://documentation.nokia.com/aces/sr/23-7-1/titles/security.html).

```
 /configure filter match-list { ip-prefix-list "SNMP-Source" prefix 192.168.10.30/32 }
 /configure filter match-list { ip-prefix-list "SSH-Sources" prefix 10.10.100.10/32 }
 /configure filter match-list { ip-prefix-list "SSH-Sources" prefix 172.16.20.0/24 }
 /configure system security cpm-filter ip-filter { admin-state enable }
 /configure system security cpm-filter ip-filter { entry 100 description "SSH Access" }
 /configure system security cpm-filter ip-filter { entry 100 match protocol tcp }
 /configure system security cpm-filter ip-filter { entry 100 match src-ip ip-prefix-list "SSH-Sources" }
 /configure system security cpm-filter ip-filter { entry 100 match dst-port eq 22 }
 /configure system security cpm-filter ip-filter { entry 100 action accept }
 /configure system security cpm-filter ip-filter { entry 200 description "SNMP Access" }
 /configure system security cpm-filter ip-filter { entry 200 match protocol udp }
 /configure system security cpm-filter ip-filter { entry 200 match src-ip ip-prefix-list "SNMP-Source" }
 /configure system security cpm-filter ip-filter { entry 200 match dst-port eq 161 }
 /configure system security cpm-filter ip-filter { entry 200 action accept }
 /configure system security cpm-filter ip-filter { entry 1000 log 101 }
 /configure system security cpm-filter ip-filter { entry 1000 action accept }
```

An IPv6 CPM filter can be configured as below:

```
 /configure filter match-list { ipv6-prefix-list "EBGP-v6-PEERS" prefix 2001:a8::4/127 }
 /configure filter match-list { ipv6-prefix-list "EBGP-v6-PEERS" prefix 2013:ab33:1::54/127 }

 /configure system security cpm-filter ipv6-filter { admin-state enable }
 /configure system security cpm-filter ipv6-filter { entry 700 description "Inbound eBGP IPv6 peers" }
 /configure system security cpm-filter ipv6-filter { entry 700 match next-header tcp }
 /configure system security cpm-filter ipv6-filter { entry 700 match src-ip ipv6-prefix-list "EBGP-v6-PEERS" }
 /configure system security cpm-filter ipv6-filter { entry 700 match dst-port eq 179 }
 /configure system security cpm-filter ipv6-filter { entry 700 action accept }
 
 /configure system security cpm-filter ipv6-filter { entry 750 description "Outbound eBGP IPv6 peers" }
 /configure system security cpm-filter ipv6-filter { entry 750 match next-header tcp }
 /configure system security cpm-filter ipv6-filter { entry 750 match src-ip ipv6-prefix-list "EBGP-v6-PEERS" }
 /configure system security cpm-filter ipv6-filter { entry 750 match src-port eq 179 }
 /configure system security cpm-filter ipv6-filter { entry 750 action accept }
```

CPM filter entry statistics can be see using the below command:

```
A:admin@SR1# show system security cpm-filter ip-filter entry 1000

===============================================================================
CPM IP Filter Entry
===============================================================================
Entry Id           : 1000
Description        : (Not Specified)
-------------------------------------------------------------------------------
Filter Entry Match Criteria :
-------------------------------------------------------------------------------
Log Id             : 101
Src. IP            : n/a
Src. Port          : n/a
Dst. IP            : n/a
Dest. Port         : n/a
Protocol           : none               Dscp               : Undefined
ICMP Type          : Undefined          ICMP Code          : Undefined
Fragment           : Off                Option-present     : Off
IP-Option          : n/a                Multiple Option    : Off
TCP-syn            : Off                TCP-ack            : Off
Action             : Forward
Match Router ID    : n/a
Dropped pkts       : 0                  Forwarded pkts     : 0
===============================================================================
```

# Management Access Filter (MAF)

The CPM CPU uses Management Access Filter (MAF) filters to perform filtering that applies to both traffic from the line cards directed to the CPM CPU as well as traffic from the management Ethernet port. Separate configuration is required for IPv4 and IPv6 packet matching conditions. Prefix lists can be used for groups of IP addresses.

In this example, we have 3 entries. The 3rd entry will log and accept all unmatched packets after the first 2 entries.

In order to avoid getting locked out of the router while configuring Management Access Filters, it is recommended to use `commit confirm` when making changes.

For more details on MAF, visit [SROS MAF Documentation](https://documentation.nokia.com/aces/sr/23-7-1/books/system-management/security-system-management.html#ai9exj5ybh).

```
  /configure system security management-access-filter ip-filter { default-action drop }
  /configure system security management-access-filter ip-filter { entry 100 description "Permit SSH Prefix" }
  /configure system security management-access-filter ip-filter { entry 100 action accept }
  /configure system security management-access-filter ip-filter { entry 100 match router-instance "management" }
  /configure system security management-access-filter ip-filter { entry 100 match protocol tcp }
  /configure system security management-access-filter ip-filter { entry 100 match src-ip ip-prefix-list "SSH-Sources" }
  /configure system security management-access-filter ip-filter { entry 100 match mgmt-port cpm }
  /configure system security management-access-filter ip-filter { entry 100 match dst-port port 22 }
  /configure system security management-access-filter ip-filter { entry 200 description "Permit SNMP Prefix" }
  /configure system security management-access-filter ip-filter { entry 200 action accept }
  /configure system security management-access-filter ip-filter { entry 200 match router-instance "management" }
  /configure system security management-access-filter ip-filter { entry 200 match protocol udp }
  /configure system security management-access-filter ip-filter { entry 200 match src-ip ip-prefix-list "SNMP-Source" }
  /configure system security management-access-filter ip-filter { entry 200 match mgmt-port cpm }
  /configure system security management-access-filter ip-filter { entry 200 match dst-port port 161 }
  /configure system security management-access-filter ip-filter { entry 2000 description "Management Plane Default" }
  /configure system security management-access-filter ip-filter { entry 2000 action accept }
  /configure system security management-access-filter ip-filter { entry 2000 log-events true }
  /configure system security management-access-filter ip-filter { entry 2000 match router-instance "management" }
  /configure system security management-access-filter ip-filter { entry 2000 match mgmt-port cpm }
```

An IPv6 Management Access Filter can be configured as below:

```
    /configure system security management-access-filter ipv6-filter default-action drop
    /configure system security management-access-filter ipv6-filter entry 10 { action accept }
    /configure system security management-access-filter ipv6-filter entry 10 { match router-instance "management" }
    /configure system security management-access-filter ipv6-filter entry 10 { match next-header tcp-udp }
    /configure system security management-access-filter ipv6-filter entry 10 { match src-ip ipv6-prefix-list "EBGP-v6-PEERS" }
    /configure system security management-access-filter ipv6-filter entry 10 { match mgmt-port cpm }
    /configure system security management-access-filter ipv6-filter entry 1000 { action accept }
    /configure system security management-access-filter ipv6-filter entry 1000 { match router-instance "management" }
    /configure system security management-access-filter ipv6-filter entry 1000 { match mgmt-port cpm }

```

Management Access Filter statistics can be seen using the below command:

```
A:admin@SR1# show system security management-access-filter ip-filter entry 2000

===============================================================================
IPv4 Management Access Filter
===============================================================================
filter type    : ip
Def. Action    : deny
Admin Status   : enabled (no shutdown)
-------------------------------------------------------------------------------
Entry          : 2000
Description    : Management Plane Default
Src-ip         : undefined
Mgmt-port      : cpm
Protocol       : undefined
Dst-port       : undefined
Src-port       : undefined
Router-instance: management
Action         : permit
Log            : enabled
Matches        : 1424
===============================================================================
```

# Cflowd

Cflowd is a tool used to obtain samples of IPv4, IPv6, MPLS, and Ethernet traffic data flows through a router. 

For more details on Cflowd, visit [SROS Cflowd Documentation](https://documentation.nokia.com/aces/sr/23-7-1/books/router-configuration/cflowd.html).

```
    /configure cflowd overflow 10
    /configure cflowd active-flow-timeout 30
    /configure cflowd inactive-flow-timeout 10
    /configure cflowd sample-profile 1 { sample-rate 100 }
    /configure cflowd collector 10.10.10.2 port 5000 { description "Neighbor collector" }
    /configure cflowd collector 10.10.10.2 port 5000 { autonomous-system-type peer }
    /configure cflowd collector 10.10.10.2 port 5000 { version 8 }
    /configure cflowd collector 10.10.10.2 port 5000 { aggregation protocol-port true }
    /configure cflowd collector 10.10.10.2 port 5000 { aggregation source-destination-prefix true }
    /configure cflowd collector 10.10.10.9 port 2000 { description "v9collector" }
    /configure cflowd collector 10.10.10.9 port 2000 { template-set mpls-ip }
    /configure cflowd collector 10.10.10.9 port 2000 { version 9 }
```

Cflowd is then enabled on the interface.

```
/configure router "Base" interface "To-Peering-LAN" cflowd-parameters { sampling unicast type interface }
```

Cflowd status can be seen using the below command:

```
A:admin@SR1# show cflowd status

===============================================================================
Cflowd Status
===============================================================================
Cflowd Admin Status  : Enabled
Cflowd Oper Status   : Enabled
Cflowd Export Mode   : Automatic
Active Flow Timeout  : 30 seconds
---snip---

Active Flows         : 0
Dropped Flows        : 0
Total Pkts Rcvd      : 0
Total Pkts Dropped   : 0
Overflow Events      : 0
                                         Raw Flow Counts  Aggregate Flow Counts
Flows Created                                          0                      0
Flows Matched                                          0                      0
Flows Flushed                                          0                      0

==============================================================================
Sample Profile Info
==============================================================================
Profile Id     Sample Rate
------------------------------------------------------------------------------
    1                  100

===============================================================================
Version Info
===============================================================================
Version Status                   Sent                 Open               Errors
-------------------------------------------------------------------------------
    5   Disabled                    0                    0                    0
    8   Enabled                     0                    0                    0
    9   Enabled                     0                    0                    0
   10   Disabled                    0                    0                    0
===============================================================================
```

# RPKI

SROS supports RPKI for BGP prefix origin validation.

In this example, we are configuring a RPKI session with an external server and then enabling prefix origin validation under the BGP group. We are also configuring BGP to not use any routes whose origin is invalid.

For more details on RPKI implementation, visit [SROS RPKI Documentation](https://documentation.nokia.com/aces/sr/23-7-1/books/unicast-routing-protocols/bgp-unicast-routing-protocols.html#d497e11185).

```
    /configure router "Base" origin-validation rpki-session 172.31.1.2 { admin-state enable }
    /configure router "Base" origin-validation rpki-session 172.31.1.2 { local-address 10.10.1.4 }
    /configure router "Base" origin-validation rpki-session 172.31.1.2 { port 8282 }
    /configure router "Base" bgp group "eBGP-Peering" { origin-validation ipv4 true }
    /configure router "Base" bgp group "eBGP-Peering" { origin-validation ipv6 true }
    /configure router "Base" bgp best-path-selection { origin-invalid-unusable true }
```

RPKI session status can be seen using the below command:

```
A:admin@SR1# show router origin-validation rpki-session detail

===============================================================================
RPKI Session Information
===============================================================================
IP Address         : 172.31.1.2
Description        : (Not Specified)
-------------------------------------------------------------------------------
Port               : 8282               Oper State         : connect
Uptime             : 0d 00:00:00        Flaps              : 0
Active IPv4 Records: 0                  Active IPv6 Records: 0
Admin State        : Up                 Local Address      : 10.10.1.4
Hold Time          : 600                Refresh Time       : 300
Stale Route Time   : 3600               Connect Retry      : 120
Serial ID          : 0                  Session ID         : 0
===============================================================================
No. of Sessions    : 1
===============================================================================
```

# Flowspec

FlowSpec is a standardized method for using BGP to distribute traffic flow specifications (flow routes) throughout a network. FlowSpec is supported for both IPv4 and IPv6.

For more details on Flowspec implementation, visit [SROS Flowspec Documentation](https://documentation.nokia.com/aces/sr/23-7-1/books/unicast-routing-protocols/bgp-unicast-routing-protocols.html#ai9exj5yj3).

```
    /configure router "Base" bgp neighbor "192.168.0.3" { family ipv4 true }
    /configure router "Base" bgp neighbor "192.168.0.3" { family ipv6 true }
    /configure router "Base" bgp neighbor "192.168.0.3" { family flow-ipv4 true }
    /configure router "Base" bgp neighbor "192.168.0.3" { family flow-ipv6 true }
    /configure router "Base" bgp neighbor "192.168.0.4" { family ipv4 true }
    /configure router "Base" bgp neighbor "192.168.0.4" { family ipv6 true }
    /configure router "Base" bgp neighbor "192.168.0.4" { family flow-ipv4 true }
    /configure router "Base" bgp neighbor "192.168.0.4" { family flow-ipv6 true }

    /configure filter ip-filter "FSPEC-filter" default-action accept
    /configure filter ip-filter "FSPEC-filter" filter-id 99
    /configure filter ip-filter "FSPEC-filter" embed { flowspec offset 1000 }
    /configure filter ip-filter "FSPEC-filter" embed { flowspec offset 1000 router-instance "Base" }

    /configure router "Base" interface "To-Peering-LAN" ingress { filter ip "FSPEC-filter" }
```

BGP flowspec routes can be seen using the `show router bgp routes flow-ipv4`. Filter details can be seen using the below command:

```
A:admin@SR1# show filter ip "FSPEC-filter"

===============================================================================
IP Filter
===============================================================================
Filter Id           : 99                           Applied        : Yes
Scope               : Template                     Def. Action    : Forward
Type                : Normal
Shared Policer      : Off
System filter       : Unchained
Radius Ins Pt       : n/a
CrCtl. Ins Pt       : n/a
RadSh. Ins Pt       : n/a
PccRl. Ins Pt       : n/a
Entries             : 0
Description         : (Not Specified)
Filter Name         : FSPEC-filter
-------------------------------------------------------------------------------
Filter Match Criteria : IP
-------------------------------------------------------------------------------
No Match Criteria Found
===============================================================================
```

# uRPF

Unicast reverse path forwarding check (uRPF) helps to mitigate problems that are caused by the introduction of malformed or forged (spoofed) IP source addresses into a network by discarding IP packets that lack a verifiable IP source address. uRPF is supported for both IPv4 and IPv6 on network and access. 

For more details on uRPF, visit [SROS uRPF Documentation](https://documentation.nokia.com/aces/sr/23-7-1/books/router-configuration/ip-router-configuration.html#d10e138).

```
    /configure router "Base" interface "To-Peering-LAN" ipv4 { urpf-check mode loose }
    /configure router "Base" interface "To-Peering-LAN" ipv6 { urpf-check mode loose }
```

# ACL

ACL filter policies, also referred to as Access Control Lists (ACLs) or just ‟filters”, are sets of ordered rule entries specifying packet match criteria and actions to be performed to a packet upon a match. Filter policies are created with a unique filter ID and filter name. After the filter policy is created, the policy must then be associated with interfaces or services.

For more details on ACL, visit [SROS ACL Documentation](https://documentation.nokia.com/aces/sr/23-7-1/books/router-configuration/filter-policies-router-configuration.html).

```
/configure filter match-list port-list "AS7xx-Ports" { port 179 }
/configure filter match-list port-list "AS7xx-Ports" range start 30000 end 64000 { }
/configure filter ip-filter "AS700-ALLOW" filter-id 700
/configure filter ip-filter "AS700-ALLOW" entry 10 { match protocol tcp }
/configure filter ip-filter "AS700-ALLOW" entry 10 { match src-ip ip-prefix-list "SSH-Sources" }
/configure filter ip-filter "AS700-ALLOW" entry 10 { match dst-ip ip-prefix-list "SNMP-Source" }
/configure filter ip-filter "AS700-ALLOW" entry 10 { action accept }

/configure filter ipv6-filter "AS-IPv6-ALLOW" filter-id 800
/configure filter ipv6-filter "AS-IPv6-ALLOW" entry 10 { match next-header tcp }
/configure filter ipv6-filter "AS-IPv6-ALLOW" entry 10 { match src-ip ipv6-prefix-list "EBGP-v6-PEERS" }
/configure filter ipv6-filter "AS-IPv6-ALLOW" entry 10 { match src-port port-list "AS7xx-Ports" }
/configure filter ipv6-filter "AS-IPv6-ALLOW" entry 10 { action accept }
/configure filter ipv6-filter "AS-IPv6-ALLOW" entry 20 { match next-header tcp }
/configure filter ipv6-filter "AS-IPv6-ALLOW" entry 20 { match dst-ip ipv6-prefix-list "EBGP-v6-PEERS" }
/configure filter ipv6-filter "AS-IPv6-ALLOW" entry 20 { match dst-port port-list "AS7xx-Ports" }
/configure filter ipv6-filter "AS-IPv6-ALLOW" entry 20 { action accept }
```

The filters are then applied to the interface.

```
    /configure router "Base" interface "To-Peering-LAN" ingress { filter ip "AS700-ALLOW" }
    /configure router "Base" interface "To-Peering-LAN" ingress { filter ipv6 "AS-IPv6-ALLOW" }
```

ACL statistics can be seen using the below command:

```
A:admin@SR1# show filter ip 700

===============================================================================
IP Filter
===============================================================================
Filter Id           : 700                          Applied        : No
Scope               : Template                     Def. Action    : Drop
---snip---
Filter Name         : AS700-ALLOW
-------------------------------------------------------------------------------
Filter Match Criteria : IP
-------------------------------------------------------------------------------
Entry               : 10
Description         : (Not Specified)
Log Id              : n/a
Src. IP             : ip-prefix-list "SSH-Sources"
Src. Port           : n/a
Dest. IP            : ip-prefix-list "SNMP-Source"
Dest. Port          : n/a
---snip---
Egress PBR          : Disabled
Primary Action      : Forward
Ing. Matches        : 0 pkts
Egr. Matches        : 0 pkts
```

Overall resource level usage for ACL can be seen using the `tools dump resource-usage system all | match 'ACL|Total` command.

# Rate Limiting DDoS traffic using ACL

ACL policies can be used to rate limit NTP, DNS or other types of common DDoS packet types. In this example, we are rate limiting NTP and DNS packets based on UDP, packet length, ports and destination IP.

```
/configure filter match-list ip-prefix-list "Core-IP" { prefix 172.16.20.0/24 }
/configure filter ip-filter "AS700-ALLOW" type packet-length
/configure filter ip-filter "AS700-ALLOW" entry 20 { match protocol udp }
/configure filter ip-filter "AS700-ALLOW" entry 20 { match dst-ip ip-prefix-list "Core-IP" }
/configure filter ip-filter "AS700-ALLOW" entry 20 { match port eq 123 }
/configure filter ip-filter "AS700-ALLOW" entry 20 { match packet-length gt 600 }
/configure filter ip-filter "AS700-ALLOW" entry 20 { action accept }
/configure filter ip-filter "AS700-ALLOW" entry 20 { action rate-limit pir 1000 }
/configure filter ip-filter "AS700-ALLOW" entry 30 { match protocol udp }
/configure filter ip-filter "AS700-ALLOW" entry 30 { match dst-ip ip-prefix-list "Core-IP" }
/configure filter ip-filter "AS700-ALLOW" entry 30 { match port eq 53 }
/configure filter ip-filter "AS700-ALLOW" entry 30 { match packet-length gt 600 }
/configure filter ip-filter "AS700-ALLOW" entry 30 { action accept }
/configure filter ip-filter "AS700-ALLOW" entry 30 { action rate-limit pir 1000 }
```

# Redirecting suspicious traffic using ACL

ACL policies can be used to redirect suspicious DDoS packets to a scrubbing device.  This is achieved using Policy-based Routing (PBR) and Policy-based Forwarding (PBF) actions under ACL context.

In this example, we are re-directing packets based on source or destination ip match to a different next-hop.

```
/configure filter match-list ip-prefix-list "Core-IP" { prefix 172.16.20.0/24 }
/configure filter ip-filter " pbr-nh-1 " filter-id 788
/configure filter ip-filter "pbr-nh-1" entry 10 { match src-ip ip-prefix-list "Core-IP” }
/configure filter ip-filter "pbr-nh-1" entry 10 { action forward next-hop nh-ip address 172.19.20.3 }
/configure filter ip-filter "pbr-nh-1" entry 20 { match dst-ip ip-prefix-list “Core-IP" }
/configure filter ip-filter "pbr-nh-1" entry 20 { action forward next-hop nh-ip indirect true }
/configure filter ip-filter "pbr-nh-1" entry 20 { action forward next-hop nh-ip address 192.168.40.3 }

/configure router "Base" interface "To-Peering-LAN" ingress { filter ip "pbr-nh-1" }
```

# PBR - Policy Based Routing

SR OS-based routers support configuring of IPv4 and IPv6 redirect policies. Redirect policies allow specifying multiple redirect target destinations and defining status check test methods used to validate the ability for a destination to receive redirected traffic.

For more details on PBR implementation, visit [SROS PBR Documentation](https://documentation.nokia.com/aces/sr/23-7-1/books/router-configuration/filter-policies-router-configuration.html#ai9exj5xo0

```
 /configure filter redirect-policy "FIREWALL-V4" admin-state enable
 /configure filter redirect-policy "FIREWALL-V4" destination 10.200.200.0 { ping-test }
 /configure filter redirect-policy "FIREWALL-V4" destination 10.200.200.0 { ping-test interval 5 }
 /configure filter redirect-policy "FIREWALL-V4" destination 10.200.200.0 { ping-test drop-count 1 }

 
 /configure filter ip-filter "ACL_PBR_V4" filter-id 155
 /configure filter ip-filter "ACL_PBR_V4" entry 1000 { match protocol ip }
 /configure filter ip-filter "ACL_PBR_V4" entry 1000 { action forward redirect-policy "FIREWALL-V4" }
```

The filter is then applied to the interface.

```
/configure router "Base" interface "To-Peering-LAN" ingress { filter ip "ACL_PBR_V4" }
```

# QoS

SROS implements QoS with a 4-step process – Classification, Queueing, Scheduling and (Re)Marking.

For more details on QoS implementation, visit [SROS QoS Documentation](https://documentation.nokia.com/aces/sr/23-7-1/titles/qos.html).

## QoS - Classification

The below example shows sample configuration for QoS classification on a network interface in the base routing context. For QoS on interfaces inside a VRF, similar configuration can be applied on the sap-ingress and sap-egress context within configure>qos.

This example shows packet classisication using DSCP, EXP, Protocol and Destination IP.

```
/configure qos match-list ip-prefix-list “Peering-Core” prefix 10.10.10.0/24
/configure qos network "Peering-Ingress-QoS" policy-id 10
/configure qos network "Peering-Ingress-QoS" ingress { dscp be fc be }
/configure qos network "Peering-Ingress-QoS" ingress { dscp be profile out }
/configure qos network "Peering-Ingress-QoS" ingress { lsp-exp 6 fc h1 }
/configure qos network "Peering-Ingress-QoS" ingress { lsp-exp 6 profile in }
/configure qos network "Peering-Ingress-QoS" ingress { ip-criteria entry 10 match protocol tcp }
/configure qos network "Peering-Ingress-QoS" ingress { ip-criteria entry 10 match dst-ip ip-prefix-list "Peering-Core" }
/configure qos network "Peering-Ingress-QoS" ingress { ip-criteria entry 10 action fc ef }
```

The classification policy is applied to the interface.

```
/configure router "Base" interface "To-Peering-LAN" qos { network-policy "Peering-Ingress-QoS" }
```

## QoS - Queuing

In this configuration example, we are defining 4 queues.

```
/configure qos network-queue "Peering-Queue" fc be { queue 1 }
/configure qos network-queue "Peering-Queue" fc ef { queue 6 }
/configure qos network-queue "Peering-Queue" fc h1 { queue 7 }
/configure qos network-queue "Peering-Queue" fc nc { queue 8 }
/configure qos network-queue "Peering-Queue" queue 1 { mbs 50.0 }
/configure qos network-queue "Peering-Queue" queue 1 { rate pir 90 }
/configure qos network-queue "Peering-Queue" queue 1 { rate cir 10 }
/configure qos network-queue "Peering-Queue" queue 6 { cbs 70.0 }
/configure qos network-queue "Peering-Queue" queue 6 { mbs 100.0 }
/configure qos network-queue "Peering-Queue" queue 6 { rate pir 100 }
/configure qos network-queue "Peering-Queue" queue 6 { rate cir 100 }
/configure qos network-queue "Peering-Queue" queue 7 { rate pir 100 }
/configure qos network-queue "Peering-Queue" queue 7 { rate cir 100 }
/configure qos network-queue "Peering-Queue" queue 8 { rate pir 10 }
/configure qos network-queue "Peering-Queue" queue 8 { rate cir 10 }
```

The queuing policy is applied under the FP context of the line card.

```
/configure card 1 fp 1 { ingress network queue-policy "Peering-Queue" }
```

## QoS - Scheduling

The example shows a simple port-based scheduler which typically meets the requirements for a Peering network. SROS also supports Hierarchical schedulers and Slope policies.

```
/configure qos port-scheduler-policy "Peer-Scheduler" max-rate 100000000
/configure qos port-scheduler-policy "Peer-Scheduler" level 1 { rate pir 90 }
/configure qos port-scheduler-policy "Peer-Scheduler" level 1 { rate cir 10 }
/configure qos port-scheduler-policy "Peer-Scheduler" level 6 { rate pir max }
/configure qos port-scheduler-policy "Peer-Scheduler" level 6 { rate cir max }
/configure qos port-scheduler-policy "Peer-Scheduler" level 7 { rate pir max }
/configure qos port-scheduler-policy "Peer-Scheduler" level 7 { rate cir max }
/configure qos port-scheduler-policy "Peer-Scheduler" level 8 { rate pir max }
/configure qos port-scheduler-policy "Peer-Scheduler" level 8 { rate cir max }
```

The port based scheduler policy is applied to the physical port.

```
/configure port 1/1/c1/1 ethernet { egress port-scheduler-policy policy-name "Peer-Scheduler" }
```

## Qos - Remarking

Re-marking configuration is done inside the same policy as classification and is applied under the interface level.

In this example, we are re-marking DSCP and EXP.

```
/configure qos network "Peering-Ingress-QoS" egress { fc be lsp-exp-in-profile 0 }
/configure qos network "Peering-Ingress-QoS" egress { fc be lsp-exp-out-profile 0 }
/configure qos network "Peering-Ingress-QoS" egress { fc ef dscp-in-profile af41 }
/configure qos network "Peering-Ingress-QoS" egress { fc h1 lsp-exp-in-profile 6 }
```

The classification & re-marking policy is applied under the interface.

```
/configure router "Base" interface "To-Peering-LAN" qos { network-policy "Peering-Ingress-QoS" }
```

# NTP

In order to provide a complete router configuration, we will also show below the NTP configuration in SROS.

```
/configure system { time ntp admin-state enable }
/configure system { time ntp server 172.16.1.10 router-instance "Base" key-id 5 }
/configure system { time ntp server 172.16.1.10 router-instance "Base" prefer true }
/configure system { time ntp server 172.18.2.20 router-instance "Base" key-id 5 }
/configure system { time ntp authentication-key 5 key "keyvalue" }
/configure system { time ntp authentication-key 5 type message-digest }
```

The status of NTP can be seen using the `show system ntp all` command.

# System Alarms and Logging

SROS has default log-id 99 for all events and log-id 100 for events with severity Major and above. These logs can be read using the `show log log-id <id>` command.
User defined logs can be created as shown below. Log destination options are File, Memory, Console, SNMP, Netconf or Syslog.

For more details on logging, visit [SROS Log Documentation](https://documentation.nokia.com/aces/sr/23-7-1/books/system-management/event-account-logs.html).

A user defined log can be created as below:

```
/configure log log-id "33" admin-state enable
/configure log log-id "33" source { main true }
/configure log log-id "33" source { security true }
/configure log log-id "33" source { change true }
/configure log log-id "33" destination { memory max-entries 500 }
```

A syslog destination can be configured as below:

```
/configure log syslog "Syslog-server-1" address 192.168.15.190
/configure log syslog "Syslog-server-1" port 514

/configure log log-id "To-syslog" admin-state enable
/configure log log-id "To-syslog" source { main true }
/configure log log-id "To-syslog" source { security true }
/configure log log-id "To-syslog" source { change true }
/configure log log-id "To-syslog" destination { syslog "Syslog-server-1" }
```

The status of all log IDs can be using the `show log log-id` command.

# CPU and Memory consumption

CPU usage can be monitored using the `show system cpu` command.

Memory usage can be monitored using the `show system memory-pools` command.

# Optional - Configuration Groups

SROS supports the creation of configuration templates called configuration groups which can be applied at different branches in the configuration using the apply-groups command. To view the expanded configuration, use the `info inherirance` command under the branch context.

For more details on Config groups, visit [SROS Configuration Groups Documentation](https://documentation.nokia.com/sr/23-7-1/books/7x50-shared/md-cli-user/edit-configuration.html#unique_469766673).

In this example, we are creating an IS-IS configuration group for authentication and metric.

```
/configure groups group "Peer-isis" router "Base" isis 0 { interface "<Interface-to-AS.*>" }
/configure groups group "Peer-isis" router "Base" isis 0 { interface "<Interface-to-AS.*>" hello-authentication-key "mykey" }
/configure groups group "Peer-isis" router "Base" isis 0 { interface "<Interface-to-AS.*>" hello-authentication-type message-digest }
/configure groups group "Peer-isis" router "Base" isis 0 { interface "<Interface-to-AS.*>" interface-type point-to-point }
/configure groups group "Peer-isis" router "Base" isis 0 { interface "<Interface-to-AS.*>" level 2 metric 10 }
```

The configuration group can be applied to an IS-IS interface:

```
/configure router "Base" { interface "Interface-to-AS65501" }
/configure router "Base" isis 0 interface "Interface-to-AS65501" { apply-groups ["Peer-isis"] }
```

The group inherited configuration can be viewed from the interface context:

```
(pr)[/configure router "Base" isis 0 interface "Interface-to-AS65501"]
A:admin@sr101# info inheritance
    apply-groups ["Peer-isis"]
    ## inherited: from group "Peer-isis"
    hello-authentication-key "7NcYcNGWMxapfjrDQIyYNe1ZQ7HXjfY=" hash2
    ## inherited: from group "Peer-isis"
    hello-authentication-type message-digest
    ## inherited: from group "Peer-isis"
    interface-type point-to-point
    level 2 {
        ## inherited: from group "Peer-isis"
        metric 10
    }
```

# Optional - Custom CLI command using pySROS

Custom python applications can be written to run on a SROS node that implements a MicroPython interpreter (version 3.4). One example of an application is a custom CLI command. The below sample configuration shows how to configure a python script get_all_interfaces.py (that will show all VRF interfaces and in/out packets in one output) to be used as a CLI command. For sample python scripts, visit [Nokia pysros GitHub](https://github.com/nokia/pysros).

For more details on pysros configuration, visit [SROS pysros Documentation](https://documentation.nokia.com/aces/sr/23-7-1/books/system-management/python.html).

Also vist the [pySROS API guide](https://documentation.nokia.com/sr/22-10-3/pysros/index.html).

The below example shows a custom CLI command to list all interfaces of all VPRNs along with their In/Out packet counters in a table format. The script is available in [Nokia pysros GitHub Examples](https://github.com/nokia/pysros/blob/main/examples/get_all_vprn_interfaces.py).

The script is copied over to the CF3 directory.

```
/configure python python-script "get_all_interfaces" { admin-state enable }
/configure python python-script "get_all_interfaces" { urls ["cf3:\get_all_interfaces.py"] }
/configure python python-script "get_all_interfaces" { version python3 }

/configure system management-interface cli md-cli environment command-alias { alias "all-interfaces" }
/configure system management-interface cli md-cli environment command-alias { alias "all-interfaces" admin-state enable }
/configure system management-interface cli md-cli environment command-alias { alias "all-interfaces" description "show all VRF interfaces" }
/configure system management-interface cli md-cli environment command-alias { alias "all-interfaces" python-script "get_all_interfaces" }
/configure system management-interface cli md-cli environment command-alias { alias "all-interfaces" mount-point "/show" }
```

After committing the changes, logout and login again to run the custom CLI command.

```
# show all-interfaces 
====================================================================================================
All Interfaces on all VPRNs
====================================================================================================
VPRN            Interface Name  IPv4 Address    Oper Status     Port:VLAN       In-Pkts    Out-Pkts  
----------------------------------------------------------------------------------------------------
100             VPRN100         99.99.99.75     up              loopback        0          0         
150             To_Ixia         150.150.150.6   up              1/1/30:150      0          0         
150             VPRN150         150.150.150.75  up              loopback        0          0         
====================================================================================================
```
