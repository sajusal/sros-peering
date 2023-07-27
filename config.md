# Nokia SROS Peering Configuration

This page provides the basic step-by-step configuration required to set up a Nokia 7750 Service Router as a Peering router. All the required feature sets for a peering router are covered here with configuration and show examples. Most sections also provides links to Nokia documentation for further reading.

All configurations are in MD-CLI flat format. Reference chassis is 7750 SR-1 and software version is SROS 23.7R1.

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

# Configuration Groups

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

The status of the interface can be seen using the below command:

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
   fe80::f8ac:c0ff:fe01:401/64                                 PREFERRED
system                           Up        Down/Down   Network system
   -                                                           -
-------------------------------------------------------------------------------
Interfaces : 2
===============================================================================
```
