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
127.0.0.1 - - [05/May/2020:11:59:01 +0000] "GET / HTTP/1.1" 404 0 "-" "curl/7.50.3" "-"
```

## solution
OpenVPN can write the proxy connection information on a specific directory (See `[dir]` in the previous paragraph). Thus, we can read this from a local NGINX instance, that will act as a reverse proxy towards our application, and write the `X-Forwarded-For` header accordingly.

### openvpn configuration (part)
```
port-share 127.0.0.1 8443 /dev/shm/openvpn-port-share/
```

### nginx configuration
First of all, NGINX must be able to read the OpenVPN temp directory content. On the main config file:
```
user nginx root;
```
Then, our virtual host (example) can be:
```
server {
    listen 8443 ssl http2 default_server;
    listen [::]:8443 ssl http2 default_server;

    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
    ssl_dhparam /etc/ssl/certs/dhparam.pem;

    # Everything is a 404
    location / {
            header_filter_by_lua_file /etc/nginx/header_filter.lua;
            return 404;
    }
    # You may need this to prevent return 404 recursion.
    location = /404.html {
            internal;
    }
}
```
Of course from here you can proxy to the final destination.

### lua script
```lua
function readAll(file)
  local f = assert(io.open(file, "rb"))
  local content = f:read("*all")
  f:close()
  return content
end

local function isempty(s)
  return s == nil or s == ''
end

ngx.log(ngx.STDERR, 'Connection params: REMOTE_ADDR=' .. ngx.var.remote_addr .. ' REMOTE_PORT=' .. ngx.var.remote_port)

local fname = '/dev/shm/openvpn-port-share/[AF_INET]' .. ngx.var.remote_addr .. ':' .. ngx.var.remote_port

ngx.log(ngx.STDERR, 'Reading file: ' .. fname)

local fcont = readAll(fname)
fcont = fcont:gsub("%[AF_INET%]", "")

ngx.log(ngx.STDERR, 'Real user: ' ..fcont)

local hostip = fcont:sub(1, fcont:find(':')-1)

ngx.log(ngx.STDERR, 'Real Source IP: ' ..hostip)

if not isempty(hostip) then
  ngx.req.set_header("X-Forwarded-For", hostip)
end
```

### testing
NGINX logs:
```
==> /var/log/nginx/access.log <==
127.0.0.1 - - [05/May/2020:12:49:33 +0000] "GET / HTTP/1.1" 404 146 "-" "curl/7.50.3" "172.17.252.215"

==> /var/log/nginx/error.log <==
2020/05/05 12:49:33 [] 7432#7432: *3 [lua] header_filter.lua:12: Connection params: REMOTE_ADDR=127.0.0.1 REMOTE_PORT=49284, client: 127.0.0.1, server: , request: "GET / HTTP/1.1", host: "192.168.199.16:1994"
2020/05/05 12:49:33 [] 7432#7432: *3 [lua] header_filter.lua:16: Reading file: /dev/shm/openvpn-port-share/[AF_INET]127.0.0.1:49284, client: 127.0.0.1, server: , request: "GET / HTTP/1.1", host: "192.168.199.16:1994"
2020/05/05 12:49:33 [] 7432#7432: *3 [lua] header_filter.lua:21: Real user: 172.17.252.215:52940, client: 127.0.0.1, server: , request: "GET / HTTP/1.1", host: "192.168.199.16:1994"
2020/05/05 12:49:33 [] 7432#7432: *3 [lua] header_filter.lua:25: Real Source IP: 172.17.252.215, client: 127.0.0.1, server: , request: "GET / HTTP/1.1", host: "192.168.199.16:1994"
```
