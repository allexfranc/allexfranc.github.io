---
layout: default
---

<section>
  <h3>Bio</h3>
  <p>Professional Android Security Engineer. Amateur systems programmer. Expert at breaking things in both domains.</p>
  
  <p>
    <a href="https://linkedin.com/in/{{ site.linkedin }}">LinkedIn</a> | 
    <a href="https://github.com/{{ site.github }}">GitHub</a>
  </p>
</section>

## Posts ~~Recent Catastrophes~~

{% for post in site.posts %}
### [{{ post.title }}]({{ post.url }})
{{ post.date | date: "%B %d, %Y" }} - {{ post.excerpt }}
{% endfor %}