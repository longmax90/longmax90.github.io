---
layout: default
title: Tags
permalink: /tags/
---

## Browse by Tag

<ul class="taxonomy-list">
  {% assign sorted_tags = site.tags | sort %}
  {% for tag in sorted_tags %}
  <li>
    <a href="#{{ tag[0] | slugify }}">{{ tag[0] }}</a>
    <span class="taxonomy-count">({{ tag[1] | size }})</span>
  </li>
  {% endfor %}
</ul>

{% for tag in sorted_tags %}
## {{ tag[0] }}

<ul class="taxonomy-posts" id="{{ tag[0] | slugify }}">
  {% for post in tag[1] %}
  <li>
    <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%Y-%m-%d" }}</time>
    <a href="{{ post.url }}">{{ post.title }}</a>
  </li>
  {% endfor %}
</ul>
{% endfor %}
