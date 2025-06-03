---
layout: default
---

# {{site.title}}

> {{site.description}}

Professional Android developer. Amateur systems programmer. Expert at breaking things in both domains.

[LinkedIn](https://linkedin.com/in/{{ site.linkedin }}) | [GitHub](https://github.com/{{ site.github }})

## Posts ~~Recent Catastrophes~~

{% for post in site.posts %}
### [{{ post.title }}]({{ post.url }})
{{ post.date | date: "%B %d, %Y" }} - {{ post.excerpt }}
{% endfor %}