---
layout: post
title: "mitmproxy - Watch webservices at work"
---

Usually, when I find a new interesting tool, I leave a bookmark in my "Tools"-folder, forget
it and after some months I'll miss it... So I thought I start a series of blog posts of
the tools I use as a reminder for myself and of course as suggestions for the reader. It was
pretty quiet in this blog anyway :)


# mitmproxy

[`mitmproxy`](http://docs.mitmproxy.org/) ("Man-In-The-Middle proxy", 
[Github](https://github.com/mitmproxy/mitmproxy)) is a small HTTP(s)-proxy, 
that allows you to observe, inspect and manipulate requests. So every time you wonder, what you 
your application and/or services send and receive to and from other services you can startup 
`mitmproxy` and configure the proxy.

`mitmproxy` starts curses-like commandline UI. There is also [`mitmdump`](http://docs.mitmproxy.org/en/stable/mitmdump.html),
which provides "tcpdump-like functionality". In other words, `mitmproxy` is the
interactive ui, whereas `mitmdump` is a programmatic helper.

This post is a quick introduction. `mitmproxy` can do much more for you, like
manipulating request, avoid any caching, …. I suggest to [read the documentation](http://docs.mitmproxy.org/en/stable/introduction.html).

# Installation

Arch provides `mitmproxy` in its [Community repository](https://www.archlinux.org/packages/community/any/mitmproxy/).
The version is `0.15` (2015-12-05).

```bash
pacman -S mitmproxy
```

Debian has `mitmproxy` in their official repositories too

```bash
apt-get install mitmproxy
```

However, the version [0.10 in Jessie](https://packages.debian.org/jessie/mitmproxy) is obviously
older. The same for ubuntu: [Wily comes with 0.11](http://packages.ubuntu.com/wily/mitmproxy)
and there is no official package for Precise (the LTS).

So, why do I tell you about the available versions? It seems this tools is under heavy
development and the Arch-version I use has some options and probably abilities the other
versions don't have. It makes sense to always try the latest version.

# Startup

To start `mitmproxy`, just ... well, start it.

```bash
mitmproxy --port 8000
```

This will let `mitmproxy` listen on Port `8000`.

On the commandline for most tools you can set the environment variables `HTTP_PROXY`
and `HTTPS_PROXY`. For all popular browser there are proxy-switcher plugins, that
let you switch between different proxy profiles. More interesting are programmatic
clients, that are used within applications to communicate with other services. All of them
should provide a "proxy"-setting somehow somewhere. For example Guzzle let you [set
a proxy during for a request](http://docs.guzzlephp.org/en/5.3/clients.html#proxy), or as
[default during instanciation](http://docs.guzzlephp.org/en/5.3/clients.html#creating-a-client).

There are also [several ways to let `mitmproxy` watch the traffic](http://docs.mitmproxy.org/en/stable/modes.html),
if you have to handle more complex infrastructures

# Watch

You should see some requests coming in now. For example when I request [kingcrunch.eu/](http://kingcrunch.eu/)
I can see this:

![Overview](/images/mitmproxy-overview.png)

This is the overview. You can already see some useful infos, but especially you can
see _all_ requests made since startup. With the arrow keys you can now select one entry
and with `Enter` open the details. Use `Tab` to switch between Request- and Response-Infos and 
details about the connection.

![Request](/images/mitmproxy-request.png)

![Response](/images/mitmproxy-response.png)

![Details](/images/mitmproxy-details.png)

# HTTPS

You may notice, that `https://kingcrunch.eu/` should be SSL-encrypted, but `mitmproxy`
is still able to track it. To do so `mitmproxy` actually performs an
"[Mitm-Attack](https://en.wikipedia.org/wiki/Man-in-the-middle_attack)" by creating
a custom certificate for the requested domain on-the-fly. The tool creates a special 
CA-certificate for your local machine. To avoid "unsecure connection"-warnings
you can trust this certificate, but You really should keep the private key private, 
else it makes you vulnerable for _real_ mitm-attacks.

You can find the self-generated CA-certificate in `~/.mitmproxy/mitmproxy-ca-cert.*`.
There is one `.p12` for use with windows, one `.pem` for use with non-windows systems, and
one `.crt`, which is actually a `.pem`, but with a different extension for use with
android. You can also visit [mitm.it](http://mitm.it). This will show you several
download links, when the traffic is passed through the proxy. Else you'll a simple
static page from the real `mitm.it`-domain telling you, that you setup your proxy first.

In Arch installing a certificate uses [`trust`](https://www.archlinux.org/news/ca-certificates-update/).

```bash
cp ~/.mitmproxy/mitmproxy-ca-cert.pem /etc/ca-certificates/trust-source/anchors/
trust extract-compat
```

Debian makes use of the probably more widely known `update-ca-certificates`

```console
$ cp ~/.mitmproxy/mitmproxy-ca-cert.pem /usr/local/share/ca-certificates/
$ update-ca-certificates
```

In Android after downloading the certificate via http://mitm.it the OS already asks,
if it should install it.

# Summary

For me `mitmproxy` already served well. The setup is pretty simple, because HTTP-clients
from "real" clients like browser, over programmatic browser in form of libraries, to
embedded clients (think of android apps) should be able to send its traffic through a proxy.

The other (bigger) benefit is, that once you have it up and running you can see
all requests/responses made since start. When you see something strange, you don't have
to redo everything and watch logs. I also saw requests made be the application, that shouldn't
happen at all, or requests were made in the wrong order.
