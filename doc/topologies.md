Supported Topologies for Octavia load balancing

This document describes the topologies that should be supported by Octavia in
an OpenStack Neutron environment, as well as key characteristics of each
topology.

=======

# Supported

## Single node, non-routed (flat)

### Features / Faults

* Simplest topology
* Single node does all the load balancing
* No abiltiy to scale seamlessly
* Not resilient to single hardware failures

### Behavioral quirks

* "HA routing IP" is technically not necessary in this topology. (Though we
will still use it because it simplifies several aspects of management.)
* All service IPs must live on interface that participates in ARP with layer-2
network

## HA active-standby, non-routed (flat)

### Features / Faults

* Simplest HA topology
* Active node does all the load balancing, standby monitors active node and
will step in / take over if the active node fails.
* Roles should be "sticky" (ie. role changes do not automatically fail-back.)
* Ability to scale vertically by following this process:
** Destroy standby node
** Boot new, larger standby node
** Make sure configs are synchronized between nodes
** Flip-flop roles
** Destroy old active node (now standby node)
** Boot new, larger standby node
** Make sure configs are synchronized between nodes
* Resilient to single point of failure

### Behavioral quirks

* "HA routing IP" is technically not necessary in this topology. (Though we
will still use it because it simplifies several aspects of management.)
* All service IPs and HA routing IP must live on interface that participates
in ARP with layer-2 network.
* Care must be taken so that only active node responds to arp requests for
HA routing IP and all service IPs
* When fail-over happens, new active node must send gratuitous ARP for the
HA routing IP as well as all service IPs.

=======

# Anticipated
Topologies mentioned in this section are not yet possible due to limitations
of the Neutron LBaaS model or Neutron networking itself. The specific missing
features are documented below.

## Single-node, routed

### Features / Faults

Same as above, non-routed mode

### What needs to happen to support this:

* Problem of inserting static routes into neutron network (routing traffic
destined for service IP subnets to HA routing IP) needs to be solved.

## HA active-standby, routed

Same as above, non-routed mode

### What needs to happen to support this:

* Problem of inserting static routes into neutron network (routing traffic
destined for service IP subnets to HA routing IP) needs to be solved.
* When fail-over happens, new active node needs to send gratuituous arp for
only the HA routing IP.

## HA n-node active-active (routed only)

### Features / Faults

* Not technically HA unless at least 2 octavia virtual appliances are available
at all times
* Abilty to scale widely horizontally.
* Ability to auto-scale horizontally (we anticipate instrumentation will be
in place for this, but actual command and control will be handled through heat
project)
* Also ability to scale vertically following same process as HA active-standby
* Can only be done in routed mode

### What needs to happen to support this:

* Problem of inserting static routes into neutron network (routing traffic
destined for service IP subnets to HA routing IP) needs to be solved.
* Need to have flow-based "dumb" router that lives above load balancing cluster
that distributes flows to various octavia virtual appliances according to
some configured algorithm.
* Need to have very basic monitor in place for octavia virtual appliances so
that these can be removed from flow-based router config when they fail.
