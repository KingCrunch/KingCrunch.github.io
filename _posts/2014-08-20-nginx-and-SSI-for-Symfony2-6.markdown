---
layout: post
title: "Setting up SSI on nginx to work with Symfony 2.6"
description: ""
category: "development"
tags: []
---
{% include JB/setup %}

With the upcoming [Symfony](http://symfony.com/) 2.6 [SSI support](https://github.com/symfony/symfony/commit/bf140a8487a19d767b9b57102e74c842e8f0a208)
is directly built-in. For me this is pretty exciting, because it is the first merged PR, that is more than a minor
addition, or bugfix. However, setting up [nginx](http://nginx.org/) wasn't that flawless at the end. More about that later in this post.

{% highlight nginx %}
server {
    server_name domain.tld www.domain.tld;
    root /var/www/project/web;

    location / {
        # try to serve file directly, fallback to app.php
        try_files $uri /app.php$is_args$args;
    }

    location ~ ^/(app|app_dev|config)\.php(/|$) {
        fastcgi_pass unix:/var/run/php5-fpm.sock;
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param HTTPS off;
    }

    error_log /var/log/nginx/project_error.log;
    access_log /var/log/nginx/project_access.log;
}
{% endhighlight %}

The SSI-implementation behaves like the long existing [ESI-implementation](http://symfony.com/doc/current/book/http_cache.html#using-esi-in-symfony2),
which means you [have to enable it on server-side too](http://symfony.com/doc/current/cookbook/cache/varnish.html) [^1].
As long as the server doesn't tell the Symfony2-stack, that it is _able_ to handle SSI-tags (or ESI-tags),
it falls back to the InlineRenderer, that renders the response of the action directly into the document [^2].
To do this, we add a header.

In the response the application itself adds a header. The intention of this header is the exact inverse idea of the
header we send to the application: It allows the server to decide, whether, or not substitution is required. This post
only covers unconditional SSI: It tries to find and replace SSI-tags, even if there is none just to keep it simple.
However, we still don't want anybody outside to see this, so we will remove it.


{% highlight nginx %}
location ~ ^/(app|app_dev|config)\.php(/|$) {
    ssi on;

    # Other options

    fastcgi_param HTTP_SURROGATE_CAPABILITY symfony2="SSI/1.0";
    fastcgi_hide_header SURROGATE_CONTROL;
}
{% endhighlight %}

While this was as simple as it could be, now here comes the ugliness: It doesn't work! If there is a SSI-tag, it will end
up in an infinite loop...

The problem is not directly visible in our configuration, because it comes from the default `fastcgi_params`. This
file contains (beside others) this line:

{% highlight nginx %}
# [..]
fastcgi_param REQUEST_URI $request_uri;
# [..]
{% endhighlight %}

Digging further we can find something in the [nginx Manual](http://nginx.org/en/docs/http/ngx_http_core_module.html#var_request_uri),
that describes, what this means:

    $request_uri
        full original request URI (with arguments)
    [..]
    $uri
        current URI in request, normalized
        The value of $uri may change during request processing,
        e.g. when doing internal redirects, or when using index files.

The definition of "full original request URI" is interpreted pretty strict by nginx, because it _always_ and in every
case contains the initial URI. For nginx everything, that somehow influences the _used_ URI is treated as an internal
subrequest and to perform a subrequest nginx updates `$uri` and starts processing this new URI from the beginning. That
includes `rewrite` of course, `try_files` and even SSI, even if it is actually something slightly different than the
previous mentioned rewrites.

For our configuration above this means, that every SSI-subrequest back to the Symfony2-application will have the same
and identical `REQUEST_URI`-parameter. Of course the result will contain the SSI-tag again. I spent a long time
investigating the best solution to work around this, that both "works" and is not too complex. At the end I've found my
solution, that at least works. The downside is, that now URIs like `example.com/app.php/foo/bar` (--> note the
`app.php/`) doesn't work anymore.


{% highlight nginx %}
server {
    server_name domain.tld www.domain.tld;
    root /var/www/project/web;

    location / {
        set $orig_uri $uri; # <-- Remember uri during first iteration of every (sub)request
        try_files $uri /app.php$uri$is_args$args;
    }

    location ~ ^/(app|app_dev|config)\.php(/|$) {
        ssi on;

        fastcgi_pass unix:/var/run/php5-fpm.sock;
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param HTTPS off;

        fastcgi_param REQUEST_URI $orig_uri$is_args$args; # <-- Use $orig_uri instead

        fastcgi_param HTTP_SURROGATE_CAPABILITY symfony2="SSI/1.0";
        fastcgi_hide_header SURROGATE_CONTROL;
    }

    error_log /var/log/nginx/project_error.log;
    access_log /var/log/nginx/project_access.log;
}
{% endhighlight %}

I had something in mind with a named location, but I hadn't followed it any further for now.
I'll probably come back to this later.

[^1]: Read more about [Edge Architecture Specification](http://www.w3.org/TR/edge-arch), primary initiated by
      [Akamai Technologies](http://www.akamai.com).

[^2]: Although SSI itself doesn't mention to use headers to control it's
      behaviour I've decided to follow the ESI-specifications for the Symfony2-implementation too to avoid confusion.
