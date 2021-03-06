# Web server

The Netdata web server runs as `static-threaded`, i.e. with a fixed, configurable number of threads.
It uses non-blocking I/O and respects the `keep-alive` HTTP header to serve multiple HTTP requests via the same connection.

## Configuration

You can disable the web server by editing `netdata.conf` and setting:

```
[web]
    mode = none
```

With the web server enabled, you can control the number of threads and sockets with the following settings:

```
[web]
    web server threads = 4
    web server max sockets = 512
```

The default number of processor threads is `min(cpu cores, 6)`.

The `web server max sockets` setting is automatically adjusted to 50% of the max number of open files netdata is allowed to use (via `/etc/security/limits.conf` or systemd), to allow enough file descriptors to be available for data collection.

### Binding netdata to multiple ports

Netdata can bind to multiple IPs and ports, offering access to different services on each. Up to 100 sockets can be used (you can increase it at compile time with `CFLAGS="-DMAX_LISTEN_FDS=200" ./netdata-installer.sh ...`).

The ports to bind are controlled via `[web].bind to`, like this:

```
[web]
   default port = 19999
   bind to = 127.0.0.1=dashboard^SSL=optional 10.1.1.1:19998=management|netdata.conf hostname:19997=badges [::]:19996=streaming^SSL=force localhost:19995=registry *:http=dashboard unix:/tmp/netdata.sock
```

Using the above, netdata will bind to:

- IPv4 127.0.0.1 at port 19999 (port was used from `default port`). Only the UI (dashboard) and the read API will be accessible on this port. Both HTTP and HTTPS requests will be accepted.
- IPv4 10.1.1.1 at port 19998. The management API and netdata.conf will be accessible on this port.
- All the IPs `hostname` resolves to (both IPv4 and IPv6 depending on the resolved IPs) at port 19997. Only badges will be accessible on this port.
- All IPv6 IPs at port 19996. Only metric streaming requests from other netdata agents will be accepted on this port. Only encrypted streams will be allowed (i.e. slaves also need to be [configured for TLS](../../streaming).
- All the IPs `localhost` resolves to (both IPv4 and IPv6 depending the resolved IPs) at port 19996. This port will only accept registry API requests.
- All IPv4 and IPv6 IPs at port `http` as set in `/etc/services`. Only the UI (dashboard) and the read API will be accessible on this port. 
- Unix domain socket `/tmp/netdata.sock`. All requests are serviceable on this socket.

The option `[web].default port` is used when an entries in `[web].bind to` do not specify a port.

Note that the access permissions specified with the `=request type|request type|...` format are available from version 1.12 onwards. 
As shown in the example above, these permissions are optional, with the default being to permit all request types on the specified port. 
The request types are strings identical to the `allow X from` directives of the access lists, i.e. `dashboard`, `streaming`, `registry`, `netdata.conf`, `badges` and `management`. 
The access lists themselves and the general setting `allow connections from` in the next section are applied regardless of the ports that are configured to provide these services. 
The API requests are serviced as follows:
- `dashboard` gives access to the UI, the read API and badges API calls.
- `badges` gives access only to the badges API calls.
- `management` gives access only to the management API calls.

### Enabling TLS support


Netdata since version 1.16 supports encrypted HTTP connections to the web server and encryption of the data stream between a slave and a master. 
Inbound unix socket connections are unaffected, regardless of the SSL settings.  
To enable SSL, provide the path to your certificate and private key in the `[web]` section of `netdata.conf`:

```
[web]
	ssl key = /etc/netdata/ssl/key.pem
	ssl certificate = /etc/netdata/ssl/cert.pem
```

Both files must be readable by the netdata user. If any of the two files does not exist or is unreadable, Netdata falls back to HTTP.

For a master/slave connection, only the master needs these settings.

For test purposes, you can generate self-signed certificates with the following command:

```
$ openssl req -newkey rsa:2048 -nodes -sha512 -x509 -days 365 -keyout key.pem -out cert.pem
```

TIP: If you use 4096 bits for the key and the certificate, netdata will need more CPU to process the whole communication. 
rsa4096 can be until 4 times slower than rsa2048, so we recommend using 2048 bits. You can verify the difference by running

```
$ openssl speed rsa2048 rsa4096
```

#### SSL enforcement

When the certificates are defined and unless any other options are provided, a Netdata server will:
- Redirect all incoming HTTP web server requests to HTTPS. Applies to the dashboard, the API, netdata.conf and badges.
- Allow incoming slave connections to use both unencrypted and encrypted communications for streaming.
 
To change this behavior, you need to modify the `bind to` setting in the `[web]` section of `netdata.conf`.
At the end of each port definition, you can append `^SSL=force` or `^SSL=optional`. What happens with these settings differs, depending on whether the port is used for HTTP/S requests, or for streaming.

SSL setting | HTTP requests | HTTPS requests | Unencrypted Streams | Encrypted Streams 
:------:|:-----:|:-----:|:-----:|:-------- 
none | Redirected to HTTPS | Accepted | Accepted | Accepted
`force` | Redirected to HTTPS | Accepted | Denied | Accepted
`optional` | Accepted | Accepted | Accepted | Accepted

Example:

```
[web]
    bind to = *=dashboard|registry|badges|management|streaming|netdata.conf^SSL=force
```

For information how to configure the slaves to use TLS, check [securing the communication](../../streaming#securing-the-communication) in the streaming documentation.
You will find there additional details on the expected behavior for client and server nodes, when their respective SSL options are enabled.

#### SSL error

It is possible that when you start to use the Netdata with SSL some erros will be register in the logs, this happens due possible incompatibilities between the browser options related to SSL like Ciphers and TLS/SSL version and the Netdata internal configuration. The most common error would be `error:00000006:lib(0):func(0):EVP lib`. In a near future the Netdata will allow our users to change the internal configuration to avoid errors like this, but until there we are setting the most common and safety options to the communication.

### Access lists

Netdata supports access lists in `netdata.conf`:

```
[web]
	allow connections from = localhost *
	allow dashboard from = localhost *
	allow badges from = *
	allow streaming from = *
	allow netdata.conf from = localhost fd* 10.* 192.168.* 172.16.* 172.17.* 172.18.* 172.19.* 172.20.* 172.21.* 172.22.* 172.23.* 172.24.* 172.25.* 172.26.* 172.27.* 172.28.* 172.29.* 172.30.* 172.31.*
	allow management from = localhost
```

`*` does string matches on the IPs of the clients.

- `allow connections from` matches anyone that connects on the netdata port(s).
   So, if someone is not allowed, it will be connected and disconnected immediately, without reading even
   a single byte from its connection. This is a global settings with higher priority to any of the ones below.

- `allow dashboard from` receives the request and examines if it is a static dashboard file or an API call the
   dashboards do.

- `allow badges from` checks if the API request is for a badge. Badges are not matched by `allow dashboard from`.

- `allow streaming from` checks if the slave willing to stream metrics to this netdata is allowed.
   This can be controlled per API KEY and MACHINE GUID in [stream.conf](../../streaming/stream.conf).
   The setting in `netdata.conf` is checked before the ones in [stream.conf](../../streaming/stream.conf).

- `allow netdata.conf from` checks the IP to allow `http://netdata.host:19999/netdata.conf`.
   The IPs listed are all the private IPv4 addresses, including link local IPv6 addresses. Keep in mind that connections to netdata API ports are filtered by `allow connections from`. So, IPs allowed by `allow netdata.conf from` should also be allowed by `allow connections from`.

- `allow management from` checks the IPs to allow API management calls. Management via the API is currently supported for [health](../api/health/#health-management-api)

### Other netdata.conf [web] section options
setting | default | info
:------:|:-------:|:----
ses max window | `15` | See [single exponential smoothing](../api/queries/des/)
des max window | `15` | See [double exponential smoothing](../api/queries/des/)
listen backlog | `4096` | The port backlog. Check `man 2 listen`.
web files owner | `netdata` | The user that owns the web static files. Netdata will refuse to serve a file that is not owned by this user, even if it has read access to that file. If the user given is not found, netdata will only serve files owned by user given in `run as user`.
web files group | `netdata` | If this is set, Netdata will check if the file is owned by this group and refuse to serve the file if it's not.
disconnect idle clients after seconds | `60` | The time in seconds to disconnect web clients after being totally idle.
timeout for first request | `60` | How long to wait for a client to send a request before closing the socket. Prevents slow request attacks.
accept a streaming request every seconds | `0` | Can be used to set a limit on how often a master Netdata server will accept streaming requests from the slaves in a [streaming and replication setup](../../streaming)
respect do not track policy | `no` | If set to `yes`, will respect the client's browser preferences on storing cookies. 
x-frame-options response header |  | [Avoid clickjacking attacks, by ensuring that the content is not embedded into other sites](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options).
enable gzip compression | `yes` | When set to `yes`, netdata web responses will be GZIP compressed, if the web client accepts such responses. 
gzip compression strategy | `default` | Valid strategies are `default`, `filtered`, `huffman only`, `rle` and `fixed`
gzip compression level | `3` | Valid levels are 1 (fastest) to 9 (best ratio)


## DDoS protection

If you publish your netdata to the internet, you may want to apply some protection against DDoS:

1. Use the `static-threaded` web server (it is the default)
2. Use reasonable `[web].web server max sockets` (the default is)
3. Don't use all your cpu cores for netdata (lower `[web].web server threads`)
4. Run netdata with a low process scheduling priority (the default is the lowest)
5. If possible, proxy netdata via a full featured web server (nginx, apache, etc)


[![analytics](https://www.google-analytics.com/collect?v=1&aip=1&t=pageview&_s=1&ds=github&dr=https%3A%2F%2Fgithub.com%2Fnetdata%2Fnetdata&dl=https%3A%2F%2Fmy-netdata.io%2Fgithub%2Fweb%2Fserver%2FREADME&_u=MAC~&cid=5792dfd7-8dc4-476b-af31-da2fdb9f93d2&tid=UA-64295674-3)]()
