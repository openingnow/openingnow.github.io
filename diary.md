---
layout: category
title: Diary
---
{% for post in site.posts %}
{% if post.tags contains page.title %}
## <a href="{{ post.url | remove:".html" }}">{{ post.title }}</a>
{{ post.excerpt }} _ {{ post.date | date_to_string }}에 작성.
{% endif %}
{% endfor %}
