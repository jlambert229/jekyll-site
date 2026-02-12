---
layout: page
title: Series
subtitle: Posts grouped by series
---

{% assign categories = site.categories | sort %}
{% for category in categories %}
## {{ category[0] }}

{% assign sorted_posts = category[1] | sort: "date" %}
{% for post in sorted_posts %}
{{ forloop.index }}. [{{ post.title }}]({{ post.url }}) — <small>{{ post.date | date: "%B %-d, %Y" }}</small>
{% endfor %}

{% endfor %}
