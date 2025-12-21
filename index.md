---
---

Welcome to My Home Page

{% assign date = '2020-04-13T10:20:00Z' %}

- Original date - {{ date }}
- With timeago filter - {{ date | timeago }}

{% for post in site.posts %}
- [{{ post.title }}]({{ post.url }})
  {{ post.excerpt }}
{% endfor %}

  WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: egkoppel
      WORDPRESS_DB_PASSWORD: fdjfrjkrjkrjkfgdhbhbkfdjk
      WORDPRESS_DB_NAME: egkoppelwp