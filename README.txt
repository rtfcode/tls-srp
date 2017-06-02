Notes on testbed for implementing SRP on IoT and embedded systems


Apache Setup
------------

In /etc/httpd/conf/httpd.conf:

Add support for gnutls:
LoadModule gnutls_module modules/mod_gnutls.so

Add a virtual host that uses TLS-SRP:
Include conf/extra/httpd-srp.conf


Add the file httpd-srp.conf to /etc/httpd/conf/extra:
----- /etc/httpd/conf/extra/httpd-srp.conf -----
#
# When we also provide SSL we have to listen to the 
# standard HTTP port (see above) and to the HTTPS port
#
Listen 443

##
## SSL Virtual Host Context
##

<VirtualHost 127.0.0.1:443>
    ServerName "srpserver"
    LogLevel debug
    DocumentRoot "/srv/srp"
    <Directory /srv/srp>
        Options Indexes FollowSymLinks Includes
        AllowOverride None
        Require all granted
        AddType text/html .shtml
        AddOutputFilter INCLUDES .shtml
        DirectoryIndex index.shtml

    </Directory>

    GnuTLSEnable on

    # Only use SRP for key exchange
    #GnuTLSPriorities NONE:+AES-256-CBC:+AES-128-CBC:+SRP:+SHA1:+COMP-NULL:+VERS-TLS1.1:+VERS-TLS1.0:+VERS-SSL3.0
    GnuTLSPriorities NONE:+AES-256-CBC:+AES-128-CBC:+SRP:+SHA1:+COMP-NULL:+VERS-TLS1.1:+VERS-TLS1.0:+VERS-SSL3.0:+VERS-TLS1.2

    GnuTLSCertificateFile /etc/httpd/certs/cert.pem
    GnuTLSKeyFile /etc/httpd/certs/key.pem

    GnuTLSSRPPasswdFile /etc/httpd/certs/tpasswd
    GnuTLSSRPPasswdConfFile /etc/httpd/certs/tpasswd.conf
</VirtualHost>
----- end file -----


Create the directory /srv/srp and create the file index.shtml:
----- /srv/srp/index.shtml -----
<html>
	<head><title>HTTPS!!!</title></head>
	<body>
		<h1>HTTPS!!!</h1>
		user is: <!--#echo var="SSL_SRP_USER" -->
	</body>
</html>
----- end file -----


Add the srpserver host to your /etc/hosts:
127.0.0.1	localhost.localdomain	localhost srpserver


Create a dummy certificate and key for gnutls_module (doesn't start up without):
/etc/httpd/certs/cert.pem
/etc/httpd/certs/key.pem


Create the SRP tpasswd and tpasswd.conf files in /etc/httpd/certs:
srptool --create-conf /etc/httpd/certs/tpasswd.conf
srptool --passwd /etc/httpd/certs/tpasswd --passwd-conf /etc/httpd/certs/tpasswd.conf -u admin


Launch Apache with:
sudo httpd



Curl
----

Test browsing the TLS-SRP page with curl:
curl -vvv -k --tlsuser admin --tlspassword <passwd> --tlsauthtype SRP https://srpserver:443/




Stunnel Patch
-------------

Patch stunnel with the provided patch to provide SRP support.  ./configure, make


Stunnel SRP to HTTP
-------------------

Use stunnel to provide an SRP endpoint that redirects to plain HTTP.
Create file srp2http.conf:
----- srp2http.conf -----
[SRP server]
accept = 442
connect = 80
ciphers = SRP
SRPVerifier = /path/to/password.srpv
----- end file -----

Create /path/to/password.srpv:
touch /path/to/password.srpv
openssl srp -srpvfile /path/to/password.srpv -add -gn 1536 admin

Start up stunnel:
src/stunnel srp2http.conf

Test with curl:
curl -vvv -k --tlsuser admin --tlspassword <passwd> --tlsauthtype SRP https://localhost:442/



Stunnel HTTP to SRP
-------------------

Use stunnel to provide an HTTP endpoint that redirects to an SRP endpoint.
Create file http2srp.conf:
------ http2srp.conf -----
[SRP client 1]
client = yes
accept = 127.0.0.1:81
connect = srpserver:443
SRPuser = admin
SRPpass = <passwd>
----- end file -----

Start up stunnel:
src/stunnel http2srp.conf

Test with a web browser or curl:
curl -vvv http://localhost:81/


