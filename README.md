# bastille-signalproxy

This is a [Bastille](https://github.com/bastillebsd/bastille) template for running an improved TLS proxy for the [Signal messaging app](https://signal.org) using [Caddy](https://caddyserver.com) in a FreeBSD [jail](https://docs.freebsd.org/en/books/handbook/jails).

**TL;DR:**

```sh
bastille bootstrap https://github.com/cycneuramus/bastille-signalproxy
bastille template TARGET \
	cycneuramus/bastille-signalproxy \
	--arg SIGNAL_PROXY_DOMAIN="signal.yourdomain.tld" \
	--arg REDIRECT_DOMAIN="https://freebsd.org"
```

---

## What?

From Signal's [blog](https://signal.org/blog/proxy-please):

> Several countries have recently blocked Signal, leaving their residents without a trusted and safe place to communicate.
>
> To help in this situation, Signal provides a built-in censorship circumvention feature and also includes support for a simple TLS proxy that can bypass these blocks in many circumstances and let people communicate privately. [...]
>
> **Please help if you can**: To help make our circumvention efforts as effective as possible, we invite those who are willing and able to set up their own proxy servers to do so by following the instructions [here](https://github.com/signalapp/Signal-TLS-Proxy).

## Why?

Signal already provides a [default setup](https://github.com/signalapp/Signal-TLS-Proxy) for hosting a proxy. This is not a piece of custom software but rather two dockerized `nginx` instances—of which one terminates TLS while the other handles the forwarding to Signal's servers—together with `certbot` for provisioning and renewing certificates for your domain. This setup is initialized by a bunch of `sh` calls in the container entrypoints that uses sleeps and loops to hold the whole thing together, and as such leaves room for several improvements.

## How?

This repository provides an alternative way of running the proxy that, compared to the default setup, has no dependencies other than `caddy` compiled with the [l4 plug-in](https://github.com/mholt/caddy-l4). The credit here goes to [mholt](https://github.com/mholt) (author of Caddy) and his [suggested ways of improving Signal's default setup](https://github.com/net4people/bbs/issues/60#issuecomment-776106390):

> - Caddy manages certificates for you, much better than sleep() + certbot can.
> - The certificate used is not necessarily new: it can be the same one used by an existing site. [...]
> - Caddy staples OCSP responses by default. That means censors can't block OCSP servers to take a service offline. Nginx does not do this without extra config, and its stapling implementation is not good. Caddy's is state-of-the-art.
> - No external dependences are required: no Docker, no libc, no Certbot, no cron, etc. Just run Caddy: `caddy run --config caddy.json` (you will want to daemonize it, probably).
>
> [...] This config can be augmented with an actual HTTPS server config so that Caddy can both serve a website at `example.com` and a Signal proxy at `example.com` -- same TLS ServerName (SNI), same IP address, etc. -- and it can reuse the existing certificate.

This is the approach taken in this repo's [Caddy config](usr/local/etc/caddy/caddy.json). To return to [mholt](https://github.com/net4people/bbs/issues/60#issuecomment-776106390):

> What this config does is establish 3 apps:
>
> - The `http` app starts an HTTP server that serves your website, with a socket accessible only on the local machine. H2C is enabled since we'll want to allow HTTP/2 over plaintext in our case (we terminate TLS earlier, see next point.)
> - The `layer4` app runs the TCP proxy that terminates TLS, then if the next bytes are another TLS handshake, it proxies those to Signal's servers. Otherwise, it proxies the remainder of the connection to our HTTP server (over plaintext, but is localhost, so meh -- I also got this working over TLS but seemed unnecessary).
> - The `tls` app tells Caddy to manage and use a certificate for `example.com` -- replace with your actual domain name.

This setup is here modified to perform a simple redirect (`REDIRECT_DOMAIN`) for any traffic that is not intended for Signal's services.

In summary:

- This setup will act as a Signal proxy if, and only if, the incoming TLS connection is intended for Signal's services (matching by [SNI](https://www.cloudflare.com/learning/ssl/what-is-sni))
- Any other traffic will be redirected to `REDIRECT_DOMAIN`

## Example usage

```sh
bastille create signalproxy 14.1-RELEASE 10.0.0.10

bastille bootstrap https://github.com/cycneuramus/bastille-signalproxy
bastille template signalproxy \
	cycneuramus/bastille-signalproxy \
	--arg SIGNAL_PROXY_DOMAIN="signal.yourdomain.tld" \
	--arg REDIRECT_DOMAIN="https://freebsd.org"
```

This presupposes that your FreeBSD system is appropriately set up. See below for an example of how that might look like.

## Example system setup

```sh
pkg install bastille git

# start containers at boot
sysrc bastille_enable=YES

# configure zfs for bastille
sysrc -f /usr/local/etc/bastille/bastille.conf bastille_zfs_enable=YES
sysrc -f /usr/local/etc/bastille/bastille.conf bastille_zfs_zpool=zroot

# configure a new bastille0 loopback interface
sysrc cloned_interfaces+=lo1
sysrc ifconfig_lo1_name="bastille0"
service netif cloneup

# create firewall rules with NAT for bastille0 interface
# https://bastillebsd.org/getting-started
vim /etc/pf.conf

	ext_if="vtnet0" # change to your external interface

	set block-policy return
	scrub in on $ext_if all fragment reassemble
	set skip on lo

	table <jails> persist
	nat on $ext_if from <jails> to any -> ($ext_if:0)
	rdr-anchor "rdr/*"

	block in all
	pass out quick keep state
	antispoof for $ext_if inet
	pass in inet proto tcp from any to any port ssh flags S/SA keep state

# enable and start firewall
sysrc pf_enable=YES
service pf start

# make sure host can route packets from the jail
sysrc gateway_enable=YES

# bootstrap a release for use with jails
bastille bootstrap 14.1-RELEASE update
```
