---
layout: post
title: "Strings are constants too"
description: "Why constants sometimes aren't worth it, or make things worse"
category: development
tags: ["php","development"]
---
{% include JB/setup %}

In our development team, we have a (more or less strict) rule: If it's a constant value,
make a constant out of it. In many cases this makes sense, or at least increase clarity,
or readability.

{% highlight php %}
class Constant {
    const HOUR = 3600;
    const DEFAULT_TIMEOUT = 120;
}
{% endhighlight %}

(Aside: We haven't left the "classes for everything"-paradigm yet)

As you can see both values have at least a small semantic value and either increase
readability (`HOUR`), or may change over time. But sometimes during code-reviews I
see comments like (simplified example)

{% highlight php %}
date('H');
// Constant string: Make a constant out of it
{% endhighlight %}

Lets assume I make a (class-)constant out of it. May it change some time in the future?
Quite sure no. Can it make anything clearer? If you are aware of [the manual](http://php.net/function.date)
(hopefully you are) probably not. Does it increase readability? In this case
it can make things even _worse_ (maybe not that obvious at first glance)

{% highlight php %}
use Foo\Bar\Constant;
date(Constant::HOUR);
{% endhighlight %}

So why is this worse? It is longer, what isn't bad on it's own, but just unnecessary,
it has a reference to a (otherwise) unrelated class, what is also acceptable,  and it
_decreases_ clarity, because it's name doesn't point out, whether, or not it makes use
of leading zeros and if it is in 12-hour- or 24-hour-format. The solution would be
something like

{% highlight php %}
class Constant {
    const HOUR_12H_WITHOUT_LEADING_ZEROS = 'g';
    const HOUR_12H_WITH_LEADING_ZEROS = 'h';
    const HOUR_24H_WITHOUT_LEADING_ZEROS = 'G';
    const HOUR_24H_WITH_LEADING_ZEROS = 'H';
}
{% endhighlight %}

Now remember the other `date`-related formatting characters. Or think of
combining them... Doesn't sound fun anymore, does it? What about "weekday"? Does that
tell you, if it's a numeric value, the name, or the shortened name? That sounds like it
will end up in a huge bunch of constants with unnecessary long names for something you
can read in the official, public available manual.

Whenever I read "That could be a constant" I usually think "Well, a constant value is a
constant too". Sometimes there is simply no good reason to substitute constant values with a
constant. Using `3600` as as "timestamp"-ish parameter should be clear to every
developer. If you are concerned, that somebody can misunderstand that, try `60*60` instead.
If you have a constant `DEFAULT_TIMEOUT` you maybe use it once, because for other connections
you may use different defaults. Is it worth it, to create this separation between the value
and the use of it? If you want to change it, you'll probably not look at the constant first,
but at the call, where it is used, which is always an indirection. Now you want to use a
different timeout for this single connection, so you'll remove the reference to the constant
anyway and maybe even add a new constant `DEFAULT_TIMEOUT_XY_CONNECTION`.

Think, before you scream for constants. Not every constant is helpful. Some are at best
superfluous. But having all this indirections -- because it wont end with a single superfluous
constant -- can get really distracting.
