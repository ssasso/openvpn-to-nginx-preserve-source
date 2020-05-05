# openvpn-to-nginx-preserve-source
LUA configuration for NGINX to be used together with OpenVPN port-share

## intro
OpenVPN has a config flag that allows to expose a WebServer (on HTTPS) on the same (TCP) port used for the VPN.
Basically, OpenVPN will proxy the request to the WebServer.

From OpenVPN 2.4 man page:
```
--port-share host port [dir]
When run in TCP server mode, share the OpenVPN port with another application, such as an HTTPS server.
If OpenVPN senses a connection to its port which is using a non-OpenVPN protocol, it will proxy the connection to the server at host:port.
Currently only designed to work with HTTP/HTTPS, though it would be theoretically possible to extend to other protocols such as ssh.

[dir] specifies an optional directory where a temporary file with name N containing content C will be dynamically generated for each proxy connection,
where N is the source IP:port of the client connection and C is the source IP:port of the connection to the proxy receiver.
This directory can be used as a dictionary by the proxy receiver to determine the origin of the connection.
Each generated file will be automatically deleted when the proxied connection is torn down.
```

However, when proxying to the final WebServer destination, the connection will have the OpenVPN host IP as source address (127.0.0.1 if the webserver runs on the same host):
```
127.0.0.1:49268 - - [05/May/2020:11:59:01 +0000] "GET / HTTP/1.1" 404 0 "-" "curl/7.50.3" "-"
```

## solution
OpenVPN can write the proxy connection information on a specific directory (See `[dir]` in the previous paragraph). Thus, we can read this from a local NGINX instance, that will act as a reverse proxy towards our application, and write the `X-Forwarded-For` header accordingly.
