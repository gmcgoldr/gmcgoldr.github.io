---
layout: default
title: Writing
is_home: true
---

<div class="index-header">
  <p class="eyebrow">Notes &amp; essays</p>
  <h1>Working things out.</h1>
  <p>Explorations in machine learning, software, and the ideas underneath.</p>
</div>

<ul class="post-list">
{% for post in site.posts %}
  {% assign post_heading = post.content | split: '
' | first | remove_first: '# ' | strip %}
  {% assign post_title = post_heading | default: post.title %}
  <li class="post-item">
    <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%b %-d, %Y" }}</time>
    <h2><a href="{{ post.url | relative_url }}">{{ post_title | escape }}</a></h2>
    <p>{{ post.content | markdownify | split: '<p>' | slice: 1, 1 | join: '' | split: '</p>' | first | strip_html | normalize_whitespace | truncatewords: 32 | escape_once }}</p>
  </li>
{% endfor %}
</ul>
