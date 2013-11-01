---
layout: home
---
{% include JB/setup %}

{% for post in site.posts %}
  <div class="item">
    <h1 class="title"><a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></h1>
    <p class="info">{{ post.date | date_to_long_string }}</p>
    {{ post.content }}
  </div>
{% endfor %}