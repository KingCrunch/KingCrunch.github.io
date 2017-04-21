---
layout: post
title: "Understood Symfony environments"
---

I can imagine this blog post will leave developers with the feeling, that
they read something very obvious. That's reasonable. From a specific point of view at
least. But my work life with (in the meantime) _many_ different "Symfony developers"
showed me, that I am by far not alone.

I thought I understood [Symfony Environments](http://symfony.com/doc/current/configuration/environments.html).
I used to think the environments represent the different stages an application gets deployed
to, or is used in. For example `test`, `acceptance`, `staging`, `production`, `preview`, but also 
unit tests. It always seemed a little bit flawed to me. Why does the 
[Symfony standard edition](https://github.com/symfony/symfony-standard)
only provides frontcoller for `dev` (`app_dev.php`) and `prod` (`app.php`), but not for 
`test` by default? Why should I use the `test`-environment for both the test-server and the 
unit tests [2]? Why do _I_ need to take care to setup the webservers correctly only to point to the
right front-controller (environment)? And others. Small details, that felt like either I, or the
entire community missed something.

And then it hit me. I got it in the exact moment _I_ told somebody else "Even `prod`
should be runnable on your local machine without incidents".

You indeed _should_ run _every_ stage with the `prod`-environment. The environments are not about 
the stage an application is in, but about the behaviour it should show. Deploying an application
on the server used for acceptance tests means, that I want to show, how it will look like in 
production. Of course it should _behave_ like production. Exactly. In every single detail.

On the other hand during development, testing, or for example -- as mentioned in the 
[Symfony documentation](http://symfony.com/doc/current/configuration/environments.html#creating-a-new-environment) --
benchmarking you _want_ different behaviours. You want something different from production, 
that helps you to achieve your goal. Benchmarking? You need a profiler. Development? You want 
some more insight into the applications architectural details. Testing? You need something 
you can verify your assertions against.

Still, there are differences between the stages. I'll just summarize some points
I imagine colleagues will ask me.

> Whats about ... Logging?

Questions: Do you _really_ need specific log levels during acceptance, when
you don't need them in production? Do you actually look at them? Do you really
need debug logs on the test server? Why and what do you debug there, what you cannot
debug on your development platform? But that aside. It's just that maybe different
log levels on _some_ servers are not worth the hassle to handle different environments.
Or even doesn't any benefit at all. I've seen teams, that never look at the logs on the
test server, but frequently changed the config on production.

If you still need them, consider introducing a `log_level` parameter in `parameters.yml`. 
As a side effect you can change the log-level of deployed applications without re-deployment
and even on production, when and if needed. Maybe introduce multiple parameters for different
loggers. I see this frequently with java applications and property-files. I guess it's not
too bad.

> Whats about ... APIs?

Integrating external APIs into an application is just integrating another external system.
You thought the database is part of the application? It isn't. It is something the application
uses [1]. The connection settings for the database are already in the default `parameters.yml`, 
so the settings for every other external should be there. As a side effect you prevent yourself 
from _accidentally_ accessing production APIs from your development platform just because you 
hardcoded the URLs into the `config_*.yml` and did something wrong with the environment.

> Whats about ... Mails?

Of course you don't want to send mails from your test server to your costumers. It happened
to me once and theoretically I ordered a "test"-vaccum-cleaner. But to be fair, the bug I
reported was about my email address, thus I guess they just needed an example email
address to work with. I'll dedicate a separate post for this.

At the end: Instead of saying "the difference between the stages should be minimal", you
should say "There should be no difference at all". You only need one environment for every
regular case. For every special, which goes beyond the functionality of the applications itself,
you still need environments. But that are one? Or maybe two?



[1] You can say, that the database is something the applications "owns" and "controls" in a 
    certain way, but it is not _within_ the application and there may be situations, were
    others (DBA?) access them. It's not "yours", it's just something somebody allowed you
    to use.

[2] Even the [documentation](http://symfony.com/doc/current/configuration/environments.html#executing-an-application-in-different-environments)
    mentions
    
> The `test` environment is used when writing functional tests and is not accessible in 
> the browser directly via a front controller. In other 
> words, unlike the other environments, there is no `app_test.php` front controller file.

    Yeah, I did it wrong all the time.
