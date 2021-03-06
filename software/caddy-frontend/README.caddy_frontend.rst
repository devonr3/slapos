==============
Caddy Frontend
==============

Frontend system using Caddy, based on apache-frontend software release, allowing to rewrite and proxy URLs like myinstance.myfrontenddomainname.com to real IP/URL of myinstance.

Caddy Frontend works using the master instance / slave instance design.  It means that a single main instance of Caddy will be used to act as frontend for many slaves.

Software type
=============

Caddy frontend is available in 4 software types:
  * ``default`` : The standard way to use the Caddy frontend configuring everything with a few given parameters
  * ``custom-personal`` : This software type allow each slave to edit its Caddy configuration file
  * ``default-slave`` : XXX
  * ``custom-personal-slave`` : XXX


About frontend replication
==========================

Slaves of the root instance are sent as a parameter to requested frontends which will process them. The only difference is that they will then return the would-be published information to the root instance instead of publishing it. The root instance will then do a synthesis and publish the information to its slaves. The replicate instance only use 5 type of parameters for itself and will transmit the rest to requested frontends.

These parameters are :

  * ``-frontend-type`` : the type to deploy frontends with. (default to 2)
  * ``-frontend-quantity`` : The quantity of frontends to request (default to "default")
  * ``-frontend-i-state``: The state of frontend i
  * ``-frontend-config-i-foo``: Frontend i will be requested with parameter foo
  * ``-frontend-software-release-url``: Software release to be used for frontends, default to the current software release
  * ``-sla-i-foo`` : where "i" is the number of the concerned frontend (between 1 and "-frontend-quantity") and "foo" a sla parameter.

for example::

  <parameter id="-frontend-quantity">3</parameter>
  <parameter id="-frontend-type">custom-personal</parameter>
  <parameter id="-frontend-2-state">stopped</parameter>
  <parameter id="-sla-3-computer_guid">COMP-1234</parameter>
  <parameter id="-frontend-software-release-url">https://lab.nexedi.com/nexedi/slapos/raw/someid/software/caddy-frontend/software.cfg</parameter>


will request the third frontend on COMP-1234. All frontends will be of software type ``custom-personal``. The second frontend will be requested with the state stopped

*Note*: the way slaves are transformed to a parameter avoid modifying more than 3 lines in the frontend logic.

**Important NOTE**: The way you ask for slave to a replicate frontend is the same as the one you would use for the software given in "-frontend-quantity". Do not forget to use "replicate" for software type. XXXXX So far it is not possible to do a simple request on a replicate frontend if you do not know the software_guid or other sla-parameter of the master instance. In fact we do not know yet the software type of the "requested" frontends. TO BE IMPLEMENTED

XXX Should be moved to specific JSON File

Extra-parameter per frontend with default::

  ram-cache-size = 1G
  disk-cache-size = 8G

How to deploy a frontend server
===============================

This is to deploy an entire frontend server with a public IPv4.  If you want to use an already deployed frontend to make your service available via ipv4, switch to the "Example" parts.

First, you will need to request a "master" instance of Caddy Frontend with:

  * A ``domain`` parameter where the frontend will be available
  * A ``public-ipv4`` parameter to state which public IPv4 will be used

like::

  <?xml version='1.0' encoding='utf-8'?>
  <instance>
   <parameter id="domain">moulefrite.org</parameter>
   <parameter id="public-ipv4">xxx.xxx.xxx.xxx</parameter>
  </instance>

Then, it is possible to request many slave instances (currently only from slapconsole, UI doesn't work yet) of Caddy Frontend, like::

  instance = request(
    software_release=caddy_frontend,
    partition_reference='frontend2',
    shared=True,
    partition_parameter_kw={"url":"https://[1:2:3:4]:1234/someresource"}
  )

Those slave instances will be redirected to the "master" instance, and you will see on the "master" instance the associated proper directives of all slave instances.

Finally, the slave instance will be accessible from: https://someidentifier.moulefrite.org.

About SSL and SlapOS Master Zero Knowledge
==========================================

**IMPORTANT**: One Caddy can not serve more than one specific SSL site and be compatible with obsolete browser (i.e.: IE8). See http://wiki.apache.org/httpd/NameBasedSSLVHostsWithSNI

SSL keys and certificates are directly send to the frontend cluster in order to follow zero knowledge principle of SlapOS Master.

*Note*: Until master partition or slave specific certificate is uploaded each slave is served with fallback certificate.  This fallback certificate is self signed, does not match served hostname and results with lack of response on HTTPs.

Obtaining CA for KeDiFa
-----------------------

KeDiFa uses caucase and so it is required to obtain caucase CA certificate used to sign KeDiFa SSL certificate, in order to be sure that certificates are sent to valid KeDiFa.

The easiest way to do so is to use caucase.

On some secure and trusted box which will be used to upload certificate to master or slave frontend partition install caucase https://pypi.org/project/caucase/

Master and slave partition will return key ``kedifa-caucase-url``, so then create and start a ``caucase-updater`` service::

  caucase-updater \
    --ca-url "${kedifa-caucase-url}" \
    --cas-ca "${frontend_name}.caucased.ca.crt" \
    --ca "${frontend_name}.ca.crt" \
    --crl "${frontend_name}.crl"

where ``frontend_name`` is a frontend cluster to which you will upload the certificate (it can be just one slave).

Make sure it is automatically started when trusted machine reboots: you want to have it running so you can forget about it. It will keep KeDiFa's CA certificate up to date when it gets renewed so you know you are still talking to the same service as when you previously uploaded the certificate, up to the original upload.

Master partition
----------------

After requesting master partition it will return ``master-key-generate-auth-url`` and ``master-key-upload-url``.

Doing HTTP GET on ``master-key-generate-auth-url`` will return authentication token, which is used to communicate with ``master-key-upload-url``. This token shall be stored securely.

By doing HTTP PUT to ``master-key-upload-url`` with appended authentication token it is possible to upload PEM bundle of certificate, key and any accompanying CA certificates to the master.

Example sessions is::

  request(...)

  curl -g -X GET --cacert "${frontend_name}.ca.crt" --crlfile "${frontend_name}.crl" master-key-generate-auth-url
  > authtoken

  cat certificate.pem key.pem ca-bundle.pem > master.pem

  curl -g -X PUT --cacert "${frontend_name}.ca.crt" --crlfile "${frontend_name}.crl" --data-binary @master.pem master-key-upload-url+authtoken

This replaces old request parameters:

 * ``apache-certificate``
 * ``apache-key``
 * ``apache-ca-certificate``

(*Note*: They are still supported for backward compatibility, but any value send to the ``master-key-upload-url`` will supersede information from SlapOS Master.)

Slave partition
---------------

After requesting slave partition it will return ``key-generate-auth-url`` and ``key-upload-url``.

Doing HTTP GET on ``key-generate-auth-url`` will return authentication token, which is used to communicate with ``key-upload-url``. This token shall be stored securely.

By doing HTTP PUT to ``key-upload-url`` with appended authentication token it is possible to upload PEM bundle of certificate, key and any accompanying CA certificates to the master.

Example sessions is::

  request(...)

  curl -g -X GET --cacert "${frontend_name}.ca.crt" --crlfile "${frontend_name}.crl" key-generate-auth-url
  > authtoken

  cat certificate.pem key.pem ca-bundle.pem > master.pem

  curl -g -X PUT --cacert "${frontend_name}.ca.crt" --crlfile "${frontend_name}.crl" --data-binary @master.pem key-upload-url+authtoken

This replaces old request parameters:

 * ``ssl_crt``
 * ``ssl_key``
 * ``ssl_ca_crt``

(*Note*: They are still supported for backward compatibility, but any value send to the ``key-upload-url`` will supersede information from SlapOS Master.)


How to have custom configuration in frontend server - XXX - to be written
=========================================================================

In your instance directory, you, as sysadmin, can directly edit two
configuration files that won't be overwritten by SlapOS to customize your
instance:

 * ``$PARTITION_PATH/srv/srv/apache-conf.d/apache_frontend.custom.conf``
 * ``$PARTITION_PATH/srv/srv/apache-conf.d/apache_frontend.virtualhost.custom.conf``

The first one is included in the end of the main apache configuration file.
The second one is included in the virtualhost of the main apache configuration file.

SlapOS will just create those two files for you, then completely forget them.

*Note*: make sure that the UNIX user of the instance has read access to those
files if you edit them.

Instance Parameters
===================

Master Instance Parameters
--------------------------

The parameters for instances are described at `instance-caddy-input-schema.json <instance-caddy-input-schema.json>`_.

Here some additional informations about the parameters listed, below:

domain
~~~~~~

Name of the domain to be used (example: mydomain.com). Sub domains of this domain will be used for the slave instances (example: instance12345.mydomain.com). It is then recommended to add a wild card in DNS for the sub domains of the chosen domain like::

  *.mydomain.com. IN A 123.123.123.123

Using the IP given by the Master Instance.  "domain" is a mandatory Parameter.

public-ipv4
~~~~~~~~~~~
Public ipv4 of the frontend (the one Caddy will be indirectly listening to)

port
~~~~
Port used by Caddy. Optional parameter, defaults to 4443.

plain_http_port
~~~~~~~~~~~~~~~
Port used by Caddy to serve plain http (only used to redirect to https).
Optional parameter, defaults to 8080.


Slave Instance Parameters
-------------------------

The parameters for instances are described at `instance-slave-caddy-input-schema.json <instance-slave-caddy-input-schema.json>`_.

Here some additional informations about the parameters listed, below:

path
~~~~
Only used if type is "zope".

Will append the specified path to the "VirtualHostRoot" of the zope's VirtualHostMonster.

"path" is an optional parameter, ignored if not specified.
Example of value: "/erp5/web_site_module/hosting/"

caddy_custom_https
~~~~~~~~~~~~~~~~~~
Raw Caddy configuration in python template format (i.e. write "%%" for one "%") for the slave listening to the https port. Its content will be templatified in order to access functionalities such as cache access, ssl certificates... The list is available above.

caddy_custom_http
~~~~~~~~~~~~~~~~~
Raw Caddy configuration in python template format (i.e. write "%%" for one "%") for the slave listening to the http port. Its content will be templatified in order to access functionalities such as cache access, ssl certificates... The list is available above

url
~~~
Necessary to activate cache. ``url`` of backend to use.

``url`` is an optional parameter.

Example: http://mybackend.com/myresource

domain
~~~~~~

Necessary to activate cache.

The frontend will be accessible from this domain.

``domain`` is an optional parameter.

Example: www.mycustomdomain.com

enable_cache
~~~~~~~~~~~~

Necessary to activate cache.

``enable_cache`` is an optional parameter.

Functionalities for Caddy configuration
---------------------------------------

In the slave Caddy configuration you can use parameters that will be replaced during instantiation. They should be entered as python templates parameters ex: ``%(parameter)s``:

  * ``cache_access`` : url of the cache. Should replace backend url in configuration to use the cache
  * ``access_log`` : path of the slave error log in order to log in a file.
  * ``error_log`` : path of the slave access log in order to log in a file.
  * ``certificate`` : path to the certificate


Examples
========

Here are some example of how to make your SlapOS service available through an already deployed frontend.

Simple Example (default)
------------------------

Request slave frontend instance so that https://[1:2:3:4:5:6:7:8]:1234 will be
redirected and accessible from the proxy::

  instance = request(
    software_release=caddy_frontend,
    software_type="RootSoftwareInstance",
    partition_reference='my frontend',
    shared=True,
    partition_parameter_kw={
        "url":"https://[1:2:3:4:5:6:7:8]:1234",
    }
  )


Zope Example (default)
----------------------

Request slave frontend instance using a Zope backend so that
https://[1:2:3:4:5:6:7:8]:1234 will be redirected and accessible from the
proxy::

  instance = request(
    software_release=caddy_frontend,
    software_type="RootSoftwareInstance",
    partition_reference='my frontend',
    shared=True,
    partition_parameter_kw={
        "url":"https://[1:2:3:4:5:6:7:8]:1234",
        "type":"zope",
    }
  )


Advanced example 
-----------------

Request slave frontend instance using a Zope backend, with Varnish activated,
listening to a custom domain and redirecting to /erp5/ so that
https://[1:2:3:4:5:6:7:8]:1234/erp5/ will be redirected and accessible from
the proxy::

  instance = request(
    software_release=caddy_frontend,
    software_type="RootSoftwareInstance",
    partition_reference='my frontend',
    shared=True,
    partition_parameter_kw={
        "url":"https://[1:2:3:4:5:6:7:8]:1234",
        "enable_cache":"true",
        "type":"zope",
        "path":"/erp5",
        "domain":"mycustomdomain.com",
    }
  )

Simple Example 
---------------

Request slave frontend instance so that https://[1:2:3:4:5:6:7:8]:1234 will be::

  instance = request(
    software_release=caddy_frontend,
    software_type="RootSoftwareInstance",
    partition_reference='my frontend',
    shared=True,
    software_type="custom-personal",
    partition_parameter_kw={
        "url":"https://[1:2:3:4:5:6:7:8]:1234",

        "caddy_custom_https":'
  https://www.example.com:%(https_port)s, https://example.com:%(https_port)s {
    bind %(local_ipv4)s
    tls %(certificate)s %(certificate)s

    log / %(access_log)s {combined}
    errors %(error_log)s

    proxy / https://[1:2:3:4:5:6:7:8]:1234 {
      transparent
      timeout 600s
      insecure_skip_verify
    }
  }
        "caddy_custom_http":'
  http://www.example.com:%(http_port)s, http://example.com:%(http_port)s {
    bind %(local_ipv4)s
    log / %(access_log)s {combined}
    errors %(error_log)s
  
    proxy / https://[1:2:3:4:5:6:7:8]:1234/ {
      transparent
      timeout 600s
      insecure_skip_verify
    }
  }

Simple Cache Example - XXX - to be written
------------------------------------------

Request slave frontend instance so that https://[1:2:3:4:5:6:7:8]:1234 will be::

  instance = request(
    software_release=caddy_frontend,
    software_type="RootSoftwareInstance",
    partition_reference='my frontend',
    shared=True,
    software_type="custom-personal",
    partition_parameter_kw={
        "url":"https://[1:2:3:4:5:6:7:8]:1234",
	"domain": "www.example.org",
	"enable_cache": "True",

        "caddy_custom_https":'
  ServerName www.example.org
  ServerAlias www.example.org
  ServerAlias example.org
  ServerAdmin geronimo@example.org
  SSLEngine on
  SSLProxyEngine on
  # Rewrite part
  ProxyVia On
  ProxyPreserveHost On
  ProxyTimeout 600
  RewriteEngine On
  RewriteRule ^/(.*) %(cache_access)s/$1 [L,P]',

        "caddy_custom_http":'
  ServerName www.example.org
  ServerAlias www.example.org
  ServerAlias example.org
  ServerAdmin geronimo@example.org
  SSLProxyEngine on
  # Rewrite part
  ProxyVia On
  ProxyPreserveHost On
  ProxyTimeout 600
  RewriteEngine On

  # Not using HTTPS? Ask that guy over there.
  # Dummy redirection to https. Note: will work only if https listens
  # on standard port (443).
  RewriteRule ^/(.*) %(cache_access)s/$1 [L,P],
    }
  )


Advanced example - XXX - to be written
--------------------------------------

Request slave frontend instance using custom apache configuration, willing to use cache and ssl certificates.
Listening to a custom domain and redirecting to /erp5/ so that
https://[1:2:3:4:5:6:7:8]:1234/erp5/ will be redirected and accessible from
the proxy::

  instance = request(
    software_release=caddy_frontend,
    software_type="RootSoftwareInstance",
    partition_reference='my frontend',
    shared=True,
    software_type="custom-personal",
    partition_parameter_kw={
        "url":"https://[1:2:3:4:5:6:7:8]:1234",
        "enable_cache":"true",
        "type":"zope",
        "path":"/erp5",
        "domain":"example.org",

  	"caddy_custom_https":'
  ServerName www.example.org
  ServerAlias www.example.org
  ServerAdmin example.org
  SSLEngine on
  SSLProxyEngine on
  SSLProtocol all -SSLv2 -SSLv3
  SSLCipherSuite ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:ECDH+3DES:DH+3DES:RSA+AESGCM:RSA+AES:RSA+3DES:HIGH:!aNULL:!MD5
  SSLHonorCipherOrder on
  # Use personal ssl certificates
  SSLCertificateFile %(ssl_crt)s
  SSLCertificateKeyFile %(ssl_key)s
  SSLCACertificateFile %(ssl_ca_crt)s
  SSLCertificateChainFile %(ssl_ca_crt)s
  # Configure personal logs
  ErrorLog "%(error_log)s"
  LogLevel info
  LogFormat "%%h %%l %%{REMOTE_USER}i %%t \"%%r\" %%>s %%b \"%%{Referer}i\" \"%%{User-Agent}i\" %%D" combined
  CustomLog "%(access_log)s" combined
  # Rewrite part
  ProxyVia On
  ProxyPreserveHost On
  ProxyTimeout 600
  RewriteEngine On
  # Redirect / to /index.html
  RewriteRule ^/$ /index.html [R=302,L]
  # Use cache
  RewriteRule ^/(.*) %(cache_access)s/VirtualHostBase/https/www.example.org:443/erp5/VirtualHostRoot/$1 [L,P]',

    "caddy_custom_http":'
  ServerName www.example.org
  ServerAlias www.example.org
  ServerAlias example.org
  ServerAdmin geronimo@example.org
  SSLProxyEngine on
  # Rewrite part
  ProxyVia On
  ProxyPreserveHost On
  ProxyTimeout 600
  RewriteEngine On
  # Configure personal logs
  ErrorLog "%(error_log)s"
  LogLevel info
  LogFormat "%%h %%l %%{REMOTE_USER}i %%t \"%%r\" %%>s %%b \"%%{Referer}i\" \"%%{User-Agent}i\" %%D" combined
  CustomLog "%(access_log)s" combined
  # Not using HTTPS? Ask that guy over there.
  # Dummy redirection to https. Note: will work only if https listens
  # on standard port (443).
  RewriteRule ^/(.*)$ https://%%{SERVER_NAME}%%{REQUEST_URI}',

    "ssl_key":"-----BEGIN RSA PRIVATE KEY-----
  XXXXXXX..........XXXXXXXXXXXXXXX
  -----END RSA PRIVATE KEY-----",
      "ssl_crt":'-----BEGIN CERTIFICATE-----
  XXXXXXXXXXX.............XXXXXXXXXXXXXXXXXXX
  -----END CERTIFICATE-----',
      "ssl_ca_crt":'-----BEGIN CERTIFICATE-----
  XXXXXXXXX...........XXXXXXXXXXXXXXXXX
  -----END CERTIFICATE-----',
      "ssl_csr":'-----BEGIN CERTIFICATE REQUEST-----
  XXXXXXXXXXXXXXX.............XXXXXXXXXXXXXXXXXX
  -----END CERTIFICATE REQUEST-----',
    }
  )

QUIC Protocol
=============

Note: QUIC support in Caddy is really experimental. It can result with silently having problems with QUIC connections or hanging Caddy process. So in case of QUIC error ``QUIC_NETWORK_IDLE_TIMEOUT`` or ``QUIC_PEER_GOING_AWAY`` it is required to restart caddy process.

Note: Chrome will refuse to connect to QUIC on different port then HTTPS has been served. As Caddy binds to high ports, if QUIC is wanted, the browser need to connect to high port too.

Experimental QUIC available in Caddy is not configurable. If caddy is configured to bind to HTTPS port ``${port}``, QUIC is going to be advertised on this port only. It is not possible to configure another public port in case of port rewriting.

So it is required to ``DNAT`` from ``${public IP}`` of the computer to the computer partition running caddy ``${local IP}`` with configured port::

  iptables -A DNAT -d ${public IP}/32 -p udp -m udp --dport ${port} -j DNAT --to-destination ${local IP}:${port}


Promises
========

Note that in some cases promises will fail:

 * not possible to request frontend slave for monitoring (monitoring frontend promise)
 * no slaves present (configuration promise and others)
 * no cached slave present (configuration promise and others)

This is known issue and shall be tackled soon.

KeDiFa
======

Additional partition with KeDiFa (Key Distribution Facility) is by default requested on the same computer as master frontend partition.

By adding to the request keys like ``-sla-kedifa-<key>`` it is possible to provide SLA information for kedifa partition. Eg to put it on computer ``couscous`` it shall be ``-sla-kedifa-computer_guid: couscous``.

Notes
=====

It is not possible with slapos to listen to port <= 1024, because process are
not run as root.

Solution 1 (iptables)
---------------------

It is a good idea then to go on the node where the instance is
and set some ``iptables`` rules like (if using default ports)::

  iptables -t nat -A PREROUTING -p tcp -d {public_ipv4} --dport 443 -j DNAT --to-destination {listening_ipv4}:4443
  iptables -t nat -A PREROUTING -p tcp -d {public_ipv4} --dport 80 -j DNAT --to-destination {listening_ipv4}:8080
  ip6tables -t nat -A PREROUTING -p tcp -d {public_ipv6} --dport 443 -j DNAT --to-destination {listening_ipv6}:4443
  ip6tables -t nat -A PREROUTING -p tcp -d {public_ipv6} --dport 80 -j DNAT --to-destination {listening_ipv6}:8080

Where ``{public_ipv[46]}`` is the public IP of your server, or at least the LAN IP to where your NAT will forward to, and ``{listening_ipv[46]}`` is the private ipv4 (like 10.0.34.123) that the instance is using and sending as connection parameter.

Additionally in order to access the server by itself such entries are needed in ``OUTPUT`` chain (as the internal packets won't appear in the ``PREROUTING`` chain)::

  iptables -t nat -A OUTPUT -p tcp -d {public_ipv4} --dport 443 -j DNAT --to {listening_ipv4}:4443
  iptables -t nat -A OUTPUT -p tcp -d {public_ipv4} --dport 80 -j DNAT --to {listening_ipv4}:8080
  ip6tables -t nat -A OUTPUT -p tcp -d {public_ipv6} --dport 443 -j DNAT --to {listening_ipv6}:4443
  ip6tables -t nat -A OUTPUT -p tcp -d {public_ipv6} --dport 80 -j DNAT --to {listening_ipv6}:8080

Solution 2 (network capability)
-------------------------------

It is also possible to directly allow the service to listen on 80 and 443 ports using the following command::

  setcap 'cap_net_bind_service=+ep' /opt/slapgrid/$CADDY_FRONTEND_SOFTWARE_RELEASE_MD5/go.work/bin/caddy
  setcap 'cap_net_bind_service=+ep' /opt/slapgrid/$CADDY_FRONTEND_SOFTWARE_RELEASE_MD5/parts/6tunnel/bin/6tunnel

Then specify in the master instance parameters:

 * set ``port`` to ``443``
 * set ``plain_http_port`` to ``80``

Technical notes
===============

Instantiated cluster structure
------------------------------

Instantiating caddy-frontend results with a cluster in various partitions:

 * master (the controlling one)
 * kedifa (contains kedifa server)
 * caddy-frontend-N which contains the running processes to serve sites - this partition can be replicated by ``-frontend-quantity`` parameter

So it means sites are served in `caddy-frontend-N` partition, and this partition is structured as:

 * Caddy serving the browser
 * (optional) Apache Traffic Server for caching
 * Caddy connected to the backend

Kedifa implementation
---------------------

`Kedifa <https://lab.nexedi.com/nexedi/kedifa>`_ server runs on kedifa partition.

Each `caddy-frontend-N` partition downloads certificates from the kedifa server.

Caucase (exposed by ``kedifa-caucase-url`` in master partition parameters) is used to handle certificates for authentication to kedifa server.

If ``automatic-internal-kedifa-caucase-csr`` is enabled (by default it is) there are scripts running on master partition to simulate human to sign certificates for each caddy-frontend-N node.
