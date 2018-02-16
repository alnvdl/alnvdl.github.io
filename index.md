---
layout: default
---

<div class="post-list">
    {% for post in site.posts %}
        <div class="post-item">
            <a class="post-title" href="{{ post.url }}">{{ post.title }}</a>
            <div class="post-excerpt">{{ post.excerpt }}</div>
            <div class="post-date">{{ post.date | date: "%-d %B %Y" }}</div>
        </div>
    {% endfor %}
</div>
