Virtual Appliance API

*Also known as the "back-end API"*

=====

# Table of Contents

- [Overview](#overview)
    - [lighttpd, curl and the 'Expect:' header](#lighttpd-curl-and-the-expect-header)
- [API](#api)
    - [Get appliance status](#get-appliance-status)
    - [Get services' statuses](#get-services-statuses)
    - [List all listeners](#list-all-listeners)
    - [Check listener existence](#check-listener-existence)
    - [Delete an listener](#delete-an-listener)
    - [Upload SSL certificate PEM file](#upload-ssl-certificate-pem-file)
    - [Get SSL certificate PEM file](#get-ssl-certificate-pem-file)
    - [Delete SSL certificate PEM file](#delete-ssl-certificate-pem-file)
    - [Upload listener haproxy configuration](#upload-listener-haproxy-configuration)
    - [Get listener haproxy configuration](#get-listener-haproxy-configuration)
    - [Delete listener haproxy configuration](#delete-listener-haproxy-configuration)
    - [Upload listener custom error 503 page](#upload-listener-custom-error-503-page)
    - [Get listener custom error 503 page](#get-listener-custom-error-503-page)
    - [Delete listener custom error 503 page](#delete-listener-custom-error-503-page)
    - [Upload listener stunnel configuration](#upload-listener-stunnel-configuration)
    - [Get listener stunnel configuration](#get-listener-stunnel-configuration)
    - [Delete listener stunnel configuration](#delete-listener-stunnel-configuration)

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
* 201 Created - Operation successfully resulted in the creation / processing
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

## Get appliance status

* **URL:** /status
* **Method:** GET
* **URL params:** none
* **Data params:** none
* **Success Response:**
    * Code: 200     
      Content: JSON formatted listing of various appliance statistics.
* **Error Response:**
    * none
* **Sample Call:**
```
curl -H 'Expect:' -E client_cert.pem -k https://10.0.0.2/status
```
*Response:*
```
{"network_tx":12245.6,"active_node":1,"network_rx":20521.8666666667,"haproxy_count":"2","hostname":"octavia-1.localnet","fencing_daemon_status":"OK","stunnel_count":"1","user_cpu":0.333333333333333,"system_cpu":0.233333333333333,"software_irq":0.0833333333333333,"load":["0.13","0.12","0.13"]}
```

**Notes:** The data in this request is meant to provide intelligence for an
auto-scaling orchestration controller (heat) in order to determine whether
additional (or fewer) virtual appliances are necessary to handle load. As such,
we may add additional parameters to the JSON listing above if they prove to be
useful for making these decisions.

## Get services' statuses

* **URL:** /service_status
* **Method:** GET
* **URL params:** none
* **Data params:** none
* **Success Response:**
    * Code: 200     
      Content: Human-readable listing of each service status (similar to the
      output of the 'octctl status all' command)
* **Error Response:**
    * none
* **Sample Call:**
```
curl -H 'Expect:' -E client_cert.pem -k https://10.0.0.2/service_status
```
*Response:*
```
7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c                OK
  haproxy          running (pid 27327)
635bbdc6-65cd-41fc-b879-22c02aaf8951                OK
  haproxy          running (pid 9862)
  stunnel          running (pid 9876)
```
* **Notes:**

## List all listeners

* **URL:** /listeners
* **Method:** GET
* **URL params:** none
* **Data params:** none
* **Success Response:**
    * Code: 200     
      Content: plain-text list of all listeners for which files exist on the
      current host.
* **Error Response:**
    * none
* **Sample Call:**
```
curl -H 'Expect:' -E client_cert.pem -k https://10.0.0.2/listeners
```
*Response:*
```
7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c
635bbdc6-65cd-41fc-b879-22c02aaf8951
```
* **Notes:**

## Check listener existence

* **URL:** /listeners/*:listener*
* **Method:** GET
* **URL params:**
    * *:listener* = Listener ID (ex. 7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c )
* **Data params:** none
* **Success Response:**
    * Code: 200     
      Content: OK
* **Error Response:**
    * Code: 404     
      Content: Not Found
      *none*
* **Sample Call:**
```
curl -H 'Expect:' -E client_cert.pem -k https://10.0.0.2/listeners/7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c
```
*Response:*
```
OK
```
* **Notes:** Note that this returns OK if *any* files exist for the listener
(not just if there is a valid haproxy or stunnel configuration).

## Delete an listener

* **URL:** /listeners/*:listener*
* **Method:** DELETE
* **URL params:**
    * *:listener* = Listener ID (ex. 7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c )
* **Data params:** none
* **Success Response:**
    * Code: 200     
      Content: OK (Also includes output from commands run to stop the
      listener)
* **Error Response:**
    * Code: 404     
      Content: Not Found     
      *none*
* **Sample Call:**
```
curl -k -E client_cert.pem -H 'Expect:' -v -X DELETE https://10.0.0.2/listeners/7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c
```
*Response:*
```
OK
haproxy daemon 7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c killed.
```
* **Implied actions:**
    * Stop listener
    * Delete IPs, iptables accounting rules, etc. from this appliance if
      they're no longer in use.
    * Clean up listener configuration directory.
* **Notes:**

## Upload SSL certificate PEM file

* **URL:** /listeners/*:listener*/certificates/*:filename.pem*
* **Method:** PUT
* **URL params:**
    * *:listener* = Listener ID (ex. 7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c )
    * *:filename* = PEM filename (see notes below for naming convention)
* **Data params:** Certificate data. (PEM file should be a concatenation of unencrypted RSA key, certificate and chain, in that order)
* **Success Response:**
    * Code: 201     
      Content: OK
* **Error Response:**
    * Code: 409     
      Content: No certififcate found
    * Code: 409     
      Content: No RSA key found
    * Code: 409     
      Content: Certificate and key do not match
* **Sample Call:**
```
curl -H 'Expect:' -E client_cert.pem -X PUT -T www.example.com.pem -k https://10.0.0.2/listeners/7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c/certificates/www.example.com.pem
```
*Response:*
```
OK
```
* **Implied actions:**
    * Restart stunnel and haproxy for the given listener.
* **Notes:** filename.pem should match the primary CN for which the
certificate is valid. All-caps WILDCARD should be used to replace an asterisk
in a wildcard certificate (eg. a CN of '*.example.com' should have a filename
of 'WILDCARD.example.com.pem'). Filenames must also have the .pem extension.

## Get SSL certificate PEM file

* **URL:** /listeners/*:listener*/certificates/*:filename.pem*
* **Method:** GET
* **URL params:**
    * *:listener* = Listener ID (ex. 7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c )
    * *:filename* = PEM filename (see notes below for naming convention)
* **Data params:** none
* **Success Response:**
    * Code: 200     
      Content: PEM file data
* **Error Response:**
    * Code: 404     
      Content: Not found
* **Sample Call:**
```
curl -H 'Expect:' -E client_cert.pem -k https://10.0.0.2/listeners/7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c/certificates/www.example.com.pem
```
*Response:*
```
-----BEGIN RSA PRIVATE KEY-----
MIICXQI...(cut for brevity)
-----END RSA PRIVATE KEY-----
-----BEGIN CERTIFICATE-----
MIIDEjCCAnu...(cut for brevity)
-----END CERTIFICATE-----
```
* **Implied actions:** none
* **Notes:**

## Delete SSL certificate PEM file

* **URL:** /listeners/*:listener*/certificates/*:filename.pem*
* **Method:** DELETE
* **URL params:**
    * *:listener* = Listener ID (ex. 7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c )
    * *:filename* = PEM filename (see notes below for naming convention)
* **Data params:** none
* **Success Response:**
    * Code: 200     
      Content: OK
* **Error Response:**
    * Code: 404     
      Content: Not found
* **Sample Call:**
```
curl -H 'Expect:' -E client_cert.pem -X DELETE -k https://10.0.0.2/listeners/7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c/certificates/www.example.com.pem
```
*Response:*
```
OK
```
* **Implied actions:**
    * Clean up listener configuration directory if it's now empty.
* **Notes:**

## Upload listener haproxy configuration

* **URL:** /listeners/*:listener*/haproxy
* **Method:** PUT
* **URL params:**
    * *:listener* = Listener ID (ex. 7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c )
* **Data params:** haproxy configuration file for the listener
* **Success Response:**
    * Code: 201     
      Content: OK     
      *(Also includes output from attempt to start haproxy daemon)*
* **Error Response:**
    * Code: 409     
      Content: Invalid configuration     
      *(Also includes error output from configuration check command)*
* **Sample Call:**
```
curl -k -E client_cert.pem -H 'Expect:' -v -X PUT -T haproxy.cfg https://10.0.0.2/listeners/7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c/haproxy
```
*Response:*
```
OK
Configuration file is valid
haproxy daemon for 7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c started (pid 32428)
```
* **Implied actions:**
    * Add IPs, iptables accounting rules, etc. to this appliance if they're
i     not already present.
    * Start or restart haproxy listener.
* **Notes:** The configuration uploaded is actually a template: Any occurrences of the string {MYIP} will be replaced with the appliance IP.

## Get listener haproxy configuration

* **URL:** /listeners/*:listener*/haproxy
* **Method:** GET
* **URL params:**
    * *:listener* = Listener ID (ex. 7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c )
* **Data params:** none
* **Success Response:**
    * Code: 200     
      Content: haproxy configuration file for the listener
* **Error Response:**
    * Code: 404     
      Content: Not found
* **Sample Call:**
```
curl -k -E client_cert.pem -H 'Expect:' -v https://10.0.0.2/listeners/7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c/haproxy
```
*Response:*
```
# Config file for 7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c
(cut for brevity)
```
* **Implied actions:** none
* **Notes:**

## Delete listener haproxy configuration

* **URL:** /listeners/*:listener*/haproxy
* **Method:** DELETE
* **URL params:**
    * *:listener* = Listener ID (ex. 7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c )
* **Data params:** none
* **Success Response:**
    * Code: 200     
      Content: OK (also includes output from stopping the listener)
* **Error Response:**
    * Code: 404     
      Content: Not found
* **Sample Call:**
```
curl -k -E client_cert.pem -H 'Expect:' -v -X DELETE https://10.0.0.2/listeners/7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c/haproxy
```
*Response:*
```
OK
haproxy daemon 7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c killed.
```
* **Implied actions:**
    * Stop listener
    * Delete IPs, iptables accounting rules, etc. from this appliance if
      they're no longer in use.
    * Clean up listener configuration directory if it's now empty.
* **Notes:**

## Upload listener custom error 503 page

* **URL:** /listeners/*:listener*/custom503
* **Method:** PUT
* **URL params:**
    * *:listener* = Listener ID (ex. 7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c )
* **Data params:** Custom 503 http page (note, this must include the http
headers. It is not an 'html' data file but the full http response including
headers.)
* **Success Response:**
    * Code: 201     
      Content: OK     
      *(Also includes output from attempt to restart haproxy daemon)*
* **Error Response:** none
* **Sample Call:**
```
curl -k -E client_cert.pem -H 'Expect:' -v -X PUT -T 503.http https://10.0.0.2/listeners/7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c/custom503
```
*Response:*
```
OK
Configuration file is valid
haproxy daemon for 7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c restarted (pid 1138)
```
* **Implied actions:**
    * Restart haproxy listener.
* **Notes:** Note that the custom 503 page must be < 16k in size.

## Get listener custom error 503 page

* **URL:** /listeners/*:listener*/custom503
* **Method:** GET
* **URL params:**
    * *:listener* = Listener ID (ex. 7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c )
* **Data params:** none
* **Success Response:**
    * Code: 200     
      Content: custom error 503 page for listener
* **Error Response:**
    * Code: 404     
      Content: Not found
* **Sample Call:**
```
curl -k -E client_cert.pem -H 'Expect:' -v https://10.0.0.2/listeners/7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c/custom503
```
*Response:*
```
HTTP/1.0 503 Service Unavailable
Cache-Control: no-cache
Connection: close
Content-Type: text/html

<html><body><h1>503 Service Unavailable</h1>
No server is available to handle this request, eh.
</body></html>
```
* **Implied actions:** none
* **Notes:**

## Delete listener custom error 503 page

* **URL:** /listeners/*:listener*/custom503
* **Method:** DELETE
* **URL params:**
    * *:listener* = Listener ID (ex. 7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c )
* **Data params:** none
* **Success Response:**
    * Code: 200     
      Content: OK
* **Error Response:**
    * Code: 404     
      Content: Not found
* **Sample Call:**
```
curl -k -E client_cert.pem -H 'Expect:' -v -X DELETE https://10.0.0.2/listeners/7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c/custom503
```
*Response:*
```
OK
```
* **Implied actions:** none
* **Notes:**

## Upload listener stunnel configuration

* **URL:** /listeners/*:listener*/stunnel
* **Method:** PUT
* **URL params:**
    * *:listener* = Listener ID (ex. 7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c )
* **Data params:** stunnel configuration file for the listener
* **Success Response:**
    * Code: 201      
      Content: OK
* **Error Response:**
    * Code: 409     
      Content: Invalid configuration     
      *(Also includes error output from configuration check attempt)*
* **Sample Call:**
```
curl -k -E client_cert.pem -H 'Expect:' -v -X PUT -T stunnel.conf https://10.0.0.2/listeners/7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c/stunnel
```
*Response:*
```
OK
```
* **Implied actions:**
    * Add IPs, iptables accounting rules, etc. to this appliance if they're
      not already present.
    * Start or restart stunnel and haproxy listeners.
* **Notes:** Since we don't have a way to do a non-disruptive syntax check on
the stunnel configuration file, at the present time we are required to
actually stop the existing daemon to try running a new daemon using the new
config. If this attempt fails, we attempt to roll back to the old config.
There is the possibility that one of the certificate files may have been
deleted (if other API commands were recived to do so) which may prevent a
roll-back and leave the service in a 'down' state. This should be a very rare
occurence in any case.     
.     
The configuration uploaded is actually a template: Any occurrences of the string {MYIP} will be replaced with the appliance IP.

## Get listener stunnel configuration

* **URL:** /listeners/*:listener*/stunnel
* **Method:** GET
* **URL params:**
    * *:listener* = Listener ID (ex. 7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c )
* **Data params:** none
* **Success Response:**
    * Code: 200     
      Content: stunnel configuration file for the listener
* **Error Response:**
    * Code: 404     
      Content: Not found
* **Sample Call:**
```
curl -k -E client_cert.pem -H 'Expect:' -v https://10.0.0.2/listeners/7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c/stunnel
```
*Response:*
```
; stunnel config file for 7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c
(cut for brevity)
```
* **Implied actions:** none
* **Notes:**

## Delete listener stunnel configuration

* **URL:** /listeners/*:listener*/stunnel
* **Method:** DELETE
* **URL params:**
    * *:listener* = Listener ID (ex. 7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c )
* **Data params:** none
* **Success Response:**
    * Code: 200    
      Content: OK (also includes output from stopping the listener)
* **Error Response:**
    * Code: 404     
      Content: Not found
* **Sample Call:**
```
curl -k -E client_cert.pem -H 'Expect:' -v -X DELETE https://10.0.0.2/listeners/7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c/stunnel
```
*Response:*
```
OK
haproxy daemon 7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c killed.
stunnel daemon 7e9f91eb-b3e6-4e3b-a1a7-d6f7fdc1de7c killed.
```
* **Implied actions:**
    * Stop listener
    * Delete IPs, iptables accounting rules, etc. from this appliance if
      they're no longer in use.
    * Clean up listener configuration directory if it's now empty.
* **Notes:**
