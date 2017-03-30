---
layout: page
title: "Blog"
description: "Blog."
group: navigation
---
{% include JB/setup %}

{% for post in site.posts %}
### [{{ post.title }}]({{ BASE_PATH }}{{ post.url }})
{{ post.content }}
<i>posted on {{ post.date | date: "%Y-%m-%d" }}</i>
<hr>
{% endfor %}
