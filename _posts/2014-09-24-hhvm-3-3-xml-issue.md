---
layout: post
title: "XML-issue with HHVM 3.3"
---

I tried to get [Symfony](http://symfony.com/) running on [HHVM 3.3](http://hhvm.com/blog/6239/hhvm-3-3-0),
because 3.2 caused some annoying issues. However, 3.3 didn't run out of the box
neither, because now it refused to parse DIC-XMLs. I've found the solution in
one ticket, that I cannot find anymore. I found the explanation in the
["inconsistencies"](https://github.com/facebook/hhvm/blob/c53aa2bd8ffb6e06768cbd4c9c26ff95db442369/hphp/doc/inconsistencies#L131-L134)-file
instead.

> (7) Loading of external entities in the libxml extension is disabled by default
> for security reasons. It can be re-enabled on a per-protocol basis (file, http,
> compress.zlib, etc...) with a comma-separated list in the ini setting
> hhvm.libxml.ext_entity_whitelist.

To sum it up: To get it Symfony working again, just add this to your `php.ini`

{% highlight ini %}
hhvm.libxml.ext_entity_whitelist = file
{% endhighlight %}

The security implications should be relatively small, because usually you don't
have any malicious entity-files on your local disc.

For those, who want to know: `file` is enough, because Symfony rewrites the
XML-files, so that they don't refer to the remote locations, but to local ones
within the project itself (`vendor/symfony/..`),
because the remote ones doesn't match the required versions and it prevents one
from downloading the file each and every time the container gets rebuild.
