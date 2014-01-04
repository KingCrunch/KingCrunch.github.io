---
layout: post
title: "Things you can do with a Raspberry Pi: DNS Cache server"
description: ""
category: "raspberry-pi"
tags: ["raspberry-pi"]
---
{% include JB/setup %}

If you have a Raspberry Pi and you don't know, what do to with it: Whats about a DNS-Cache-server?
There are phones, tablets, some laptops and desktop-PCs. And it doesn't need much performance. The
benefit is small, but measurable (in my case 1ms instead of minimum 20ms) and it doesn't take
much effort.

First (if not already happened) ensure, that your Raspberry Pi has static IP. Then just install `dnsmasq`

```bash
sudo apt-get install dnsmasq
```

Actually thats all. Now let all devices use this as your new DNS-server. `dnsmasq` is much more powerful
than this, but for now it's fine. For example you can let it overwrite, or just manipulate entries, or
you can extend it to act as DNS for your local network. One use case to overwrite entries is to use
this DNS-cache as adblocker. Instead of letting browser-based adblocker parse and manipulate every pages
content, we just redirect all to an alternative server (our own of course), which serves nothing.

First of all create a file `/usr/local/bin/update-adblock.sh`. Replace the IP `10.0.0.2` with the local static IP
of your raspberry pi.

```bash
#!/bin/bash
curl "http://pgl.yoyo.org/adservers/serverlist.php?hostformat=dnsmasq&showintro=0&mimetype=plaintext" | sed "s/127\.0\.0\.1/10.0.0.1/" > /etc/dnsmasq.d/adblock.conf
service dnsmasq restart
```

Of course make it executable

```bash
chmod +x /usr/local/bin/update-adblock.sh
```

Now we link this file to `/etc/cron.weekly`

```bash
sudo ln -s /usr/local/bin/update-adblock.sh /etc/cron.weekly/update-adblock
```

Most important: Let `dnsmasq` parse the newly created config file. That is by uncommenting the last line of
`/etc/dnsmasq.conf`

```
# Include a another lot of configuration options.
#conf-file=/etc/dnsmasq.more.conf
conf-dir=/etc/dnsmasq.d
```

Restart

```bash
sudo service restart dnsmasqd
```

Have fun :)

Two afterwords:

- On every Debian-based Linux you can install the package `dnsutils`, which installs (beside others)
the tool `dig`.

    ```bash
    ; <<>> DiG 9.9.3-rpz2+rl.13214.22-P2-Ubuntu-1:9.9.3.dfsg.P2-4ubuntu1 <<>> google.com
    ;; global options: +cmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 18527
    ;; flags: qr rd ra; QUERY: 1, ANSWER: 11, AUTHORITY: 0, ADDITIONAL: 0

    ;; QUESTION SECTION:
    ;google.com.			IN	A

    ;; ANSWER SECTION:
    google.com.		285	IN	A	173.194.113.128
    google.com.		285	IN	A	173.194.113.132
    google.com.		285	IN	A	173.194.113.131
    google.com.		285	IN	A	173.194.113.136
    google.com.		285	IN	A	173.194.113.130
    google.com.		285	IN	A	173.194.113.129
    google.com.		285	IN	A	173.194.113.137
    google.com.		285	IN	A	173.194.113.134
    google.com.		285	IN	A	173.194.113.133
    google.com.		285	IN	A	173.194.113.135
    google.com.		285	IN	A	173.194.113.142

    ;; Query time: 1 msec
    ;; SERVER: 10.0.0.2#53(10.0.0.2)
    ;; WHEN: Sat Jan 04 23:20:02 CET 2014
    ;; MSG SIZE  rcvd: 204
    ```

    This is how it looks on the _second_ requests, because the first one of course must
    request the regular DNS-servers. The last block tells you, that it worked and that it
    is much faster.

- Of course it's a good idea to let the Raspberry Pi use the cache itself. Edit `/etc/resolv.conf`
and _prepend_ (as the very first line) `nameserver 10.0.0.2`.
