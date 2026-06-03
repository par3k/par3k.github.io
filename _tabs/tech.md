---
layout: page
icon: fas fa-laptop-code
order: 2
title: 테크
---

{% include lang.html %}
{% assign posts = site.categories[page.title] %}

<div id="page-category">
  <h1 class="ps-lg-2">
    <i class="far fa-folder-open fa-fw text-muted"></i>
    {{ page.title }}
    <span class="lead text-muted ps-2">{{ posts | size }}</span>
  </h1>

  {% if posts.size > 0 %}
    <ul class="content ps-0">
      {% for post in posts %}
        <li class="d-flex justify-content-between px-md-3">
          <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
          <span class="dash flex-grow-1"></span>
          {% include datetime.html date=post.date class='text-muted small text-nowrap' lang=lang %}
        </li>
      {% endfor %}
    </ul>
  {% else %}
    <p class="text-muted ps-md-3 pt-3">아직 작성된 글이 없습니다.</p>
  {% endif %}
</div>
