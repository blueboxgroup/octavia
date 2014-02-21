Virtual Appliance API

*Also known as the "back-end API"*

======

# Overview

Octavia virtual appliance load balancers use a RESTful API for configuration
and control. This API is secured through the use of TLS encryption as well as
verification of client-side certificates. The certificates used for this
authentication and encryption are generated on the fly by the controller
process when the virtual appliances are spawned. Each virtual appliance has
lighttpd listening on TCP port 443 on the eth0 machine-specific IPv4 (and IPv6)
addresses (NOT the routing addresses). lighttpd has been configured only to
allow requests containing a valid client certificate, and routes all requests
to the API python back-end script.

In general, actions should be made on all load balancers in a cluster at
approximately the same time. However, actions must be at least done against
the active or primary node in the pair. (The load balancers that are either
standby or not primary will synchronize their configs periodically with the
active node.)

Beyond this, the various URLs listed below will perform different actions
based on the type of HTTP request. (Most of the URLs below allow GET, PUT or
DELETE requests.) If the virtual appliances experience problems, they will
return HTTP response codes that make sense, and attempt to roll-back the
requested operation if possible.

Typical response codes are:
* 200 OK - Operation was completed as requested.
* 201 Created - Operation successfully resulted in the creation / processingx
of a file.
* 403 Invalid Request - API script does not recognize the specified URL or
HTTP method.
* 404 Not Found - The requested file was not found.
* 409 Conflict - Usually indicates a problem with a PUT data file.
* 500 Internal Server Error - Usually indicates a permissions problem that the
API script encountered.

## lighttpd, curl and the 'Expect:' header

The current stable version of lighttpd has a problem with the PUT method, in
that certain clients will print an 'Expect:' header in the HTTP request, which
throws lighttpd off (it returns an error 419, specifically). The simple
work-around for this bug is to force curl to leave this header blank by adding
this to the command line:

```
-H 'Expect:'
```

# API

## Get server status

* URL: /status
* Method: GET
* URL params: none
* Data params: none
* Success Response:
    * Code: 200
    * Content: JSON formatted listing of various server statistics.
* Error Response:
    * none
* Sample Call:
```
curl -H 'Expect:' -E client_cert.pem -k https://199.91.169.39/status
```
* Response:
```
{"network_tx":12245.6,"active_node":1,"network_rx":20521.8666666667,"haproxy_count":"2","hostname":"lb02a.iad01.blueboxgrid.com","fencing_daemon_status":"OK","stunnel_count":"1","user_cpu":0.333333333333333,"system_cpu":0.233333333333333,"software_irq":0.0833333333333333,"load":["0.13","0.12","0.13"]}
```

Notes: The data in this request is meant to provide intelligence for an
auto-scaling orchestration controller (heat) in order to determine whether
additional (or fewer) virtual appliances are necessary to handle load. As such,
we may add additional parameters to the JSON listing above if they prove to be
useful for making these decisions.
