---
layout: default
title: "Long NGUYEN's Software Security Blog"
---

## Latest Posts

<ul class="post-list">
  {% for post in site.posts %}
  <li class="post-item">
    <p class="post-line">
      <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%Y-%m-%d" }}</time>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </p>
  </li>
  {% endfor %}
</ul>
