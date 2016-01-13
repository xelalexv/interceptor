# Purpose of this Fork

*SSLsplit* allows you to set up static forward addresses/ports, to forward connections to different destinations. When you set your HTTP(S) proxy as a forward destination, and enable pass through mode, *SSLsplit* essentially functions as a transparent HTTP(S) proxy, albeit only for HTTP and without support for proxy authentication. For HTTPS, it is necessary to first send a `CONNECT` message to the HTTP(S) proxy, and then discard the reply. 

The modifications to *SSLsplit* in branch `transparent` of this fork, fill the gap and turn *SSLsplit* into a fully transparent proxy for HTTP and HTTPS. I like to call it *The Interceptor* ;-) It finally puts an end to all the headaches we've had with our company HTTP(S) proxy...

## Usage

### Running *SSLsplit*

To run *SSLsplit* as a transparent HTTP(S) proxy, you would do for example:

```bash
sslsplit -D -P -t ./certdir/ https 0.0.0.0 8443 {HTTPS proxy} {HTTPS proxy port} http 0.0.0.0 8080 {HTTP proxy} {HTTP proxy port}
```

This instructs *SSLsplit* to accept HTTPS connections on port `8443` and HTTP connections on port `8080` (change as needed), and forward them to your HTTP(S) proxy. Replace `{HTTP(S) proxy}` and `{HTTP(S) port}` with your HTTP(S) proxy. The `-P -t ./certdir/` (with `certdir` being empty) enables unconditional HTTPS pass through. 

### IP Forwarding & Rerouting for `*:80` and `*:443`

What's missing now is a few IP tables rules that reroute your HTTP(S) traffic to *SSLsplit*. How you do that depends on your desired setup. You can run *SSLsplit* on your workstation and have it act as a purely local transparent proxy. Or you set up a dedicated machine with *SSLsplit* running and use that as the default gateway for any machine that needs transparent proxying. That's the setup we have been using quite extensively for almost a year now, and it works great for things such as `devstack`, `git` (over HTTPS), `apt-get`, `yum`, anything really.

For this centralized setup, we need to enable IP forwarding on the server so that it can act as a gateway:

```bash
sysctl -w net.ipv4.ip_forward=1
```

Then we need a few IP tables rules to reroute all traffic destined for ports `80` and `443` to `8080` and `8443` (or whatever you chose when starting *SSLsplit*) on the server instead, where *SSLsplit* is listening. A shell script to do this could look like this:

```bash
#!/bin/bash

# see: http://bramp.net/blog/2010/01/26/redirect-local-traffic-to-a-web-cache-with-iptables/

# Don't touch local traffic (localhost and internal network)
sudo iptables -t nat -A OUTPUT -o lo -j RETURN
sudo iptables -t nat -A OUTPUT --dst 127.0.0.1/8 -j RETURN
sudo iptables -t nat -A OUTPUT --dst 10.0.0.0/8 -j RETURN
# Add any other local networks here.

# reroute outgoing traffic
sudo iptables -t nat -A OUTPUT -p tcp --dport 80 -j REDIRECT --to-port 8080
sudo iptables -t nat -A OUTPUT -p tcp --dport 443 -j REDIRECT --to-port 8443

# reroute forwarded traffic
sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to 8080
sudo iptables -t nat -A PREROUTING -p tcp --dport 443 -j REDIRECT --to 8443
```

Starting *SSLsplit*, enabling IP forwarding and setting up the IP tables rules should be made persistent, of course. For starting *SSLsplit* and keeping it running, we use `supervisord`. There are occasional `SIGSEGV`s, and the restart of *SSLsplit* through `supervisord` is usually tolerated by the clients and causes no problems. 

### Authenticating Proxies

This also works with authenticating HTTP proxies, however currently only *Basic* authentication is supported (i.e. no support for *Digest* and *NTLM*). Start *SSLsplit* with the additional `-w user:password` option, to have it talk to an authenticating proxy. It will then forward all intercepted traffic to the HTTP(S) proxy with the provided credentials. Note that when using a centralized setup in this way, consequently all traffic handled by it will appear to be coming from the same user. Where this is not acceptable, a local setup per user is needed.  

### Remove Proxy Settings

You can use the *SSLsplit*-based transparent proxy in combination with explicit proxy settings. You may want to keep using explicit proxy configuration for your browser, e.g. if you already have an elaborate proxy `pac` file or automatic proxy detection. 

For sanity & peace of mind, I still think it's probably best to get rid of all proxy settings on your machine. In *Ubuntu*, type `proxy` in the dash, which suggests the network settings. Open that, and set *Network proxy* to *None*, then *Apply system wide*. If you've configured the proxy yourself anywhere, e.g. in `/etc/apt/apt.conf` for `apt-get`, remove those settings. Also check your environment settings, i.e.

```bash
env | grep -i proxy
```

should not report any proxy settings.

Happy Proxy-free Hacking!

---

Below this point, the original README...

# SSLsplit - transparent and scalable SSL/TLS interception [![Build Status](https://travis-ci.org/droe/sslsplit.svg?branch=master)](https://travis-ci.org/droe/sslsplit)
Copyright (C) 2009-2014, [Daniel Roethlisberger](//daniel.roe.ch/).  
http://www.roe.ch/SSLsplit


## Overview

SSLsplit is a tool for man-in-the-middle attacks against SSL/TLS encrypted
network connections.  Connections are transparently intercepted through a
network address translation engine and redirected to SSLsplit.  SSLsplit
terminates SSL/TLS and initiates a new SSL/TLS connection to the original
destination address, while logging all data transmitted.  SSLsplit is intended
to be useful for network forensics and penetration testing.

SSLsplit supports plain TCP, plain SSL, HTTP and HTTPS connections over both
IPv4 and IPv6.  For SSL and HTTPS connections, SSLsplit generates and signs
forged X509v3 certificates on-the-fly, based on the original server certificate
subject DN and subjectAltName extension.  SSLsplit fully supports Server Name
Indication (SNI) and is able to work with RSA, DSA and ECDSA keys and DHE and
ECDHE cipher suites.  Depending on the version of OpenSSL, SSLsplit supports
SSL 3.0, TLS 1.0, TLS 1.1 and TLS 1.2, and optionally SSL 2.0 as well.
SSLsplit can also use existing certificates of which the private key is
available, instead of generating forged ones.  SSLsplit supports NULL-prefix CN
certificates and can deny OCSP requests in a generic way.  For HTTP and HTTPS
connections, SSLsplit removes response headers for HPKP in order to prevent
public key pinning, for HSTS to allow the user to accept untrusted
certificates, and Alternate Protocols to prevent switching to QUIC/SPDY.

See the manual page sslsplit(1) for details on using SSLsplit and setting up
the various NAT engines.


## Requirements

SSLsplit depends on the OpenSSL and libevent 2.x libraries.
The build depends on GNU make and a POSIX.2 environment in `PATH`.
The optional unit tests depend on the check library.

SSLsplit currently supports the following operating systems and NAT mechanisms:

-   FreeBSD: pf rdr and divert-to, ipfw fwd, ipfilter rdr
-   OpenBSD: pf rdr-to and divert-to
-   Linux: netfilter REDIRECT and TPROXY
-   Mac OS X: pf rdr and ipfw fwd


## Installation

    make
    make test       # optional unit tests
    make install    # optional install

Dependencies are autoconfigured using pkg-config.  If dependencies are not
picked up and fixing `PKG_CONFIG_PATH` does not help, you can specify their
respective locations manually by setting `OPENSSL_BASE`, `LIBEVENT_BASE` and/or
`CHECK_BASE` to the respective prefixes.

You can override the default install prefix (`/usr/local`) by setting `PREFIX`.
For more build options see `GNUmakefile`.


## Development

SSLsplit is being developed on Github.  For bug reports, please use the Github
issue tracker.  For patch submissions, please send me pull requests.

https://github.com/droe/sslsplit


## License

SSLsplit is provided under the simplified BSD license.
SSLsplit contains components licensed under the MIT and APSL licenses.
See the respective source file headers for details.


## Credits

SSLsplit was inspired by `mitm-ssl` by Claes M. Nyberg and `sslsniff` by Moxie
Marlinspike, but shares no source code with them.

SSLsplit includes `khash.h` by Attractive Chaos.


## Contributors

The following individuals have contributed to the SSLsplit codebase by
submitting patches or pull requests, in chronological order of first
contribution:

-   Daniel Roethlisberger (@droe), main author
-   Steve Wills (@swills)
-   Landon Fuller (@landonf)
-   Wayne Jensen (@wjjensen)

See NEWS.md and `git log` for details.


