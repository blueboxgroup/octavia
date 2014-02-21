octavia
=======

Octavia is a software load balancer appliance project meant primarily to
work with OpenStack Neutron to provide a virtual appliance-based load
balancing solution.

At its core, Octavia is a linux virtual machine image and is powered by
haproxy, stunnel, and a series of glue scripts (part of this project)
which provide API and control. It can be run in several topologies, and
should be adaptable to other cloud operating systems, if desired.

This project consists of the glue scripts, configuration templates, and
other data necessary both for building the virtual appliance image and
control daemon.

Further documentation can be found in /doc
