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
[Github](https://github.com/mitmproxy/mitmproxy)) is a  small tool, that startsup a HTTP(s)-proxy, 
that allows you to observe, inspect and manipulate requests. So every time you wonder, what you 
your application and/or services send and receive to and from other services you can startup 
`mitmproxy` and configure the proxy.

`mitmproxy` starts curses-like commandline UI. There is also [`mitmdump`](http://docs.mitmproxy.org/en/stable/mitmdump.html),
which provides "tcpdump-like functionality". In other words, `mitmproxy` is the
interactive ui, whereas `mitmdump` is a programmatic helper.

# Installation

Arch provides `mitmproxy` in its [Community repository](https://www.archlinux.org/packages/community/any/mitmproxy/).
The version is `0.15` (2015-12-05).

{% highlight bash %}
pacman -S mitmproxy
{% endhighlight %}

Debian has `mitmproxy` in their official repositories too

{% highlight bash %}
apt-get install
{% endhighlight %}

However, the version [0.10 in Jessie](https://packages.debian.org/jessie/mitmproxy) is obviously
older. The same for ubuntu: [Wily comes with 0.11](http://packages.ubuntu.com/wily/mitmproxy)
and there is no official package for Precise (the LTS).

So, why do I tell you about the available versions? It seems this tools is under heavy
development and the Arch-version I use and I'll base this post on has some options
and probably abilities the other versions don't have.

# Startup

To start `mitmproxy`, just ... well, start it.

{% highlight bash %}
mitmproxy --port 8000
{% endhighlight %}

This will let `mitmproxy` listen on Port `8000`. There are many, many other options.
Just try `mitmproxy --help`. Now that the proxy is running, you only need to use
it.

On the commandline for most tools you can set the environment variables `HTTP_PROXY`
and `HTTPS_PROXY`. For all popular browser there are proxy-switcher plugins, that
let you switch between different proxy profiles. More interesting are programmatic
clients, that are used within applications to communicate with other services. All of them
should provide a "proxy"-setting somehow somewhere. For example Guzzle let you [set
a proxy during for a request](http://docs.guzzlephp.org/en/5.3/clients.html#proxy), or as
[default during instanciation](http://docs.guzzlephp.org/en/5.3/clients.html#creating-a-client).

There are also [several ways to let `mitmproxy` watch the traffic](http://docs.mitmproxy.org/en/stable/modes.html)
to watch more complex infrastructures.

# Watch

TBA!

You should see some requests coming in now. For example when I request [kingcrunch.eu/](http://kingcrunch.eu/)
I can see this:

<an image here>

Seeing the number of requests, the request uri response status at a glance is already pretty
useful, but you can go into single request. Just use the cursor keys, to select one and
enter to open the details.

<an image here>

Once it is open you can use `TAB` to switch between request, response and ???

<an image (response) here>

# HTTPS

You can intercept HTTPS connections too. How does this work? You have to _trust_
`mitmproxy`s certificate. This one is especially created for your local machine, so
you should keep the private key private, else it makes you vulnerable for _real_
mitm-attacks. After you managed to trust this certificate `mitmproxy` will create
a new certificate on-the-fly for every HTTPS you want to investigate.

You can find the self-generated CA-certificates in `~/.mitmproxy/mitmproxy-ca-cert.*`.
There is one `.p12` for use with windows, one `.pem` for use with non-windows systems, and
one `.crt`, which is actually a `.pem`, but with a different extension for use with
android. You can also visit [mitm.it](http://mitm.it). This will show you several
download links, when the traffic is passed through the proxy. Else you'll a simple
static page from the real `mitm.it`-domain telling you, that you setup your proxy first.

It's worth to mentioned, that you don't need to trust this certificates. You'll
see a browser warning every time you request a page via HTTPS, but because it's mostly
for debugging and development, it's fine I think.

In Arch installing a certificate uses [`trust`](https://www.archlinux.org/news/ca-certificates-update/).

{% highlight bash %}
cp ~/.mitmproxy/mitmproxy-ca-cert.pem /etc/ca-certificates/trust-source/anchors/
trust extract-compat
{% endhighlight %}

Debian makes use of the probably more widely known `update-ca-certificates`

{% highlight bash %}
cp ~/.mitmproxy/mitmproxy-ca-cert.pem /usr/local/share/ca-certificates/
update-ca-certificates
{% endhighlight %}

Thats it.
