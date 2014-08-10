---
layout: post
title: "Symfony2.6 comes with SSI support"
description: ""
category: "development"
tags: []
---
{% include JB/setup %}

With Symfony2.6 [SSI support](https://github.com/symfony/symfony/commit/bf140a8487a19d767b9b57102e74c842e8f0a208)
is directly built-in. For me this is pretty exciting, because it is the first merged PR,
that is more than a minor addition, or bugfix. However, setting up NGinx wasn't that flawless
at the end. The problem: NGinx doesn't update the `$request_uri` on subrequests. This means, with
the [default setup](http://symfony.com/doc/current/cookbook/configuration/web_server_configuration.html),
it ends up in an infinite loop.

```
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
```

The problem is not directly visible, because it comes from `fascgi_params`.
This file contains (beside others) this line:

```
# [..]
fastcgi_param    REQUEST_URI    $request_uri;
# [..]
```

Digging further we can find something in the [NGinx Manual](http://nginx.org/en/docs/http/ngx_http_core_module.html#var_request_uri),
that describes, what this means:

    $request_uri
        full original request URI (with arguments)
    [..]
    $uri
        current URI in request, normalized
        The value of $uri may change during request processing,
        e.g. when doing internal redirects, or when using index files.

The definition of "full original request URI" is handled pretty strict by NGinx: Even during SSI
subrequest this variable _always_ contains the URI of the _initial_ request.

And what now? It took me a _really_ long time, but at the end it's much simpler, than expected. First lets
clarify, what `$uri` is: This variable contains the _current_ URI, which may change after every (internal)
redirect, or rewrite. In o

```
server {
    server_name domain.tld www.domain.tld;
    root /var/www/project/web;

    location / {
        # try to serve file directly, fallback to app.php
        try_files $uri /app.php$uri$is_args$args; # <-- $uri new
    }

    location ~ ^/(app|app_dev|config)\.php(/|$) {
        fastcgi_pass unix:/var/run/php5-fpm.sock;
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        include fastcgi_params;
        fastcgi_param REQUEST_URI $uri; # <-- new line
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param HTTPS off;
    }

    error_log /var/log/nginx/project_error.log;
    access_log /var/log/nginx/project_access.log;
}
```

And don't forget to "activate" the SSI-handling from server side too, else Symfony2
falls back to use the inline renderer again.

```
fastcgi_param HTTP_SURROGATE_CAPABILITY symfony2="SSI/1.0";
fastcgi_hide_header     SURROGATE_CONTROL;
```
