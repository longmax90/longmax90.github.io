---
layout: default
title: Categories
permalink: /categories/
---

## Browse by Category

<ul class="taxonomy-list">
  {% assign sorted_categories = site.categories | sort %}
  {% for category in sorted_categories %}
  <li>
    <a href="#{{ category[0] | slugify }}">{{ category[0] }}</a>
    <span class="taxonomy-count">({{ category[1] | size }})</span>
  </li>
  {% endfor %}
</ul>

{% for category in sorted_categories %}
## {{ category[0] }}

<ul class="taxonomy-posts" id="{{ category[0] | slugify }}">
  {% for post in category[1] %}
  <li>
    <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%Y-%m-%d" }}</time>
    <a href="{{ post.url }}">{{ post.title }}</a>
  </li>
  {% endfor %}
</ul>
{% endfor %}
