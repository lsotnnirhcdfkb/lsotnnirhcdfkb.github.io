---
layout: default
title: Posts
permalink: /
---

# Thought Thoughts - some things

<ul>
{% for post in site.posts %}
<li><a href="{{ post.url }}">{{ post.date | date: "%d %b %Y" }}: {{ post.title }}</a></li>
{% endfor %}
</ul>

---

## About
I write things on this blog
