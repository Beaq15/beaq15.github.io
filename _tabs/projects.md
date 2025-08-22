---
title: "Projects"
layout: default
icon: fa-solid fa-users
order: 1
---

{% include lang.html %}

{% assign all_posts = site.posts | where: 'categories', 'Projects' %}

{% assign pinned = all_posts | where: 'pin', 'true' %} {% assign default = all_posts | where_exp: 'item', 'item.pin != true and item.hidden != true' %}

{% assign posts = pinned | concat: default %}

{% for post in posts %} {% include post-card.html %} {% endfor %}
{% if paginator.total_pages > 1 %} {% include post-paginator.html %} {% endif %}
</ul>
{% else %}
<p>No projects yet.</p>
{% endif %}
