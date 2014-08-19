---
layout: post
title:  "Assetic Bundle includes cache busting worker"
category: "development"
tags: ["assetic"]
---
{% include JB/setup %}

With [PR#249](https://github.com/symfony/AsseticBundle/pull/240) the [AsseticBundle](https://github.com/symfony/AsseticBundle)
(by default included in every [symfony-standard](https://github.com/symfony/symfony-standard)-based [Symfony2](http://symfony.com/)
application) supports the cache busting worker out of the box.

~~~yaml
assetic:
    workers:
        cache_busting:
            enabled: true
~~~

What happens: It will create different file names for different file _contents_, not only on the file name (At least it should, I
haden't time to try it yet). This is cool, because now you don't have to take care about cache invalidation for assetics assets yourself.

