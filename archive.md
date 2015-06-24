---
layout: page
title: Archive
summary: Archive of all blog posts.
---

## Blog Posts

{% for post in site.posts %}
  * **<time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date_to_string }}</time>** &raquo; [ {{ post.title }} ]({{ post.url }}): *{{ post.summary}}*
{% endfor %}