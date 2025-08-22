---
title: "Projects"
layout: default
icon: fa-solid fa-users
order: 1
---

{% include lang.html %}

{% assign all_posts = site.categories.Projects %}
{% assign pinned  = all_posts | where: 'pin', true %}
{% assign default = all_posts | where_exp: 'p', 'p.pin != true and p.hidden != true' %}
{% assign posts   = pinned | concat: default %}

{% include post-list.html posts=posts %}

{% if paginator and paginator.total_pages > 1 %}
  {% include post-paginator.html %}
{% endif %}
