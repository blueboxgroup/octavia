Theory of Operation
======

# Components

Octavia is made of two major components:

## Controller

This is a single daemon or process running on a controller
machine of some kind (probably the Neutron node in cluster). Its job is to
spawn virtual appliances as necessary, configure them, do periodic health
checks on them, and shut them down. In an OpenStack Neutron environment, the
controller will most likely be taking its orders from the Neutron LBaaS daemon.
It is unclear whether the 'driver' interface to Neutron LBaaS can do all the
things necessary for the controller to do in this set-up. In any case, a
Neutron LBaaS driver is the intended way for Octavia to interface with the
OpenStack cluster.

The controller communicates with the virtual appliances via a RESTful
interface on the appliances. Authentication and encryption is handled
through the use of SSL certificates. (Both client and server certificates
are used for bi-directional authentication.)

## Virtual Appliances

These are virtual machines typically run within the
compute environment which actually provide the load balancing services. More
specifically, they're bare-bones Ubuntu VMs running haproxy, stunnel, and some
glue scripts and daemons which make the back-end API work.

# Bootstrapping Virtual Appliances

The controller process spawns virtual appliances which run the actual load
balancing services. Workflow for it will generally go as follows:

1. Controller gets a command from the Neutron LBaaS interface to spin up a
new load balanced service.
2. If a virtual appliance topology (cluster) already exists that we can use,
we are done. Otherwise:
3. If we have not generated an SSL certificate for the controller, do it now.
4. Generate an SSL certificate for the new virtual appliance.
5. Communicate with NOVA to boot a virtual appliance image. In so doing the
image should be booted setting the following parameters:
	a. SSL server certificate and key appliance should use
	b. SSL client certificate the controller will use to authenticate
    c. (If debug) root SSH public key
6. Wait for the virtual appliance to boot, pinging the IP it will occupy (or
timeout)
7. Once appliance pings OK, keep hitting its status API until it returns OK
(or timeout)
8. Configure virtual appliance for the topology we're going to use.

# Configuring load balancing services

These are handled as necessary by the controller given commands it gets from
the Neutron LBaaS driver interface. Load balanced services are configured
via the virtual appliance API (back-end API) as documented in the file
in this directory. Only thing to note here:

* Back-end API commands are meant to be run synchronously-- so these will be
blocking when run.

Things to note about load balancing services:

* Each load-balanced service runs on its own service IP which will be
different than the primary interface address on any of the virtual appliances.
* If the cluster topology is not "routed mode" then the service IPs will be
added to an interface on the virtual appliance that participates in "arp" or
"neighbor discovery" with the local layer-2 network.
* If the cluster topology is in "routed mode" then the service IPs will be
added to an interface on the virtual appliances that does NOT participate in
"arp" or "neighbor discovery" with the local layer-2 network. Note that
routed-mode operation has yet to be solved for Neutron LBaaS. I anticipate
that the Octavia controller will communicate with Neutron to insert static
routes to handle this as necessary.

# Destroying Virtual Appliances

The decision to destroy a virtual appliance is going to depend a lot on the
state of the object model that Neutron LBaaS is using. If the model does not
contain a concept of a 'loadbalancer' or 'cluster' (as proposed in Feb. 2014
by Stephen Balukoff), then we should destroy the virtual load balancer(s) when
the last service on them is destroyed.

Destruction is a simple process: The controller tells Nova to destroy the virtual appliance.

# Unsolved considerations

* Should appliances have a mechanism to tell the controller they're up and
ready for service?
* The controller should periodically check the availability of all virtual
appliances. We've yet to determine where / how to do this. (We might possibly
be able to do this within the Neutron LBaaS daemon?)
* We may support altering topology from 'single' node to 'HA active/standby'
and back again. At this time, we *don't* support it due to limitations in the
Neutron LBaaS model.
