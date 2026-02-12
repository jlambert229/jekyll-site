---
layout: page
title: Posts
subtitle: All blog posts
---

{% assign posts_by_year = site.posts | group_by_exp: "post", "post.date | date: '%Y'" %}
{% for year in posts_by_year %}
## {{ year.name }}

{% for post in year.items %}
- [{{ post.title }}]({{ post.url }}) — <small>{{ post.date | date: "%B %-d, %Y" }}</small>
  {% if post.tags.size > 0 %}<br><small>{% for tag in post.tags %}<code>{{ tag }}</code>{% unless forloop.last %} {% endunless %}{% endfor %}</small>{% endif %}
{% endfor %}

{% endfor %}
