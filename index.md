---
layout: page
title: KingCrunchs!
tagline: Now in English (kind of...)
---
{% include JB/setup %}

While I abandoned my [German-speaking blog](http://www.kingcrunch.de/blog/) I've started this
one as new opportunity to collect ideas and opinions in English now. My name is Sebastian Krebs
from Berlin, Germany and I hope you'll enjoy this :)

## Blog Posts

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>
