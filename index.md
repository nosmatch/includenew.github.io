---
layout: page
title: Thinking
tagline: All about tech & life
---
{% include JB/setup %}

我是姜博，一名非纯种程序猿，这是我的个人博客，在这里你能找到一些关于Java，Mac, HDFS的记录以及技术之外的一些疯言疯语

---

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>
