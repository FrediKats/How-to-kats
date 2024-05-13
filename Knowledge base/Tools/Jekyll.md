---
title: Jekyll
---

Jekyll - это генератор статических сайт написанный на Ruby.

## excerpt

Jekyll предоставляет возможность отображать "превью" содержимого других страниц. Например, на главной странице отображать ленту из постов. Но вместо добавления всего поста использовать только первый абзац:

```
{% for post in paginator.posts %}
  <article>
    {{ post.date | date: "%B %-d, %Y" }}
    <h1>{{ post.title }}</h1>
    {{ post.excerpt }}
    <a href="{{ post.url | prepend: site.baseurl }}">Read...</a>
  </article>
{% endfor %}
```

В `_config.yml` файле можно указать сепаратор, который будет отделять превью поста: `excerpt_separator: <!--more-->`. А в самом посте его использовать:

```
Some text.

<!--more-->

Some more text. Very long text.
```

## Liquid Template