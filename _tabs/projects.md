---
title: "Projects"
layout: default
icon: fa-solid fa-users
order: 1
---

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

{% if posts and posts != empty %}
<ul class="post-list">
  {% for post in posts %}
  <li class="post-preview">
    <article>
      <h2 class="post-title">
        <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      </h2>

      {% if post.image and post.image.path %}
      <a class="post-thumbnail" href="{{ post.url | relative_url }}">
        <img src="{{ post.image.path | relative_url }}" alt="{{ post.title }}" loading="lazy">
      </a>
      {% endif %}

      {% if post.description %}
      <p class="post-desc">{{ post.description }}</p>
      {% endif %}

      <div class="post-meta">
        <time datetime="{{ post.date | date_to_xmlschema }}">
          {{ post.date | date: "%b %-d, %Y" }}
        </time>
      </div>
    </article>
  </li>
  {% endfor %}
</ul>
{% else %}
<p>No projects yet.</p>
{% endif %}
