---
layout: page
title: Archives
---

{% for cat in site.categories %}
  <h3>{{ cat[0] | replace: "-", " " |capitalize }}</h3>
  <ul>
    {% for post in cat[1] %}
      <li><a href="{{ post.url }}">{{ post.date | date: "%B %Y" }} - {{ post.title }}</a></li>
    {% endfor %}
  </ul>
{% endfor %}
