---
layout: archive
permalink: /blog/
author_profile: true
---

{% include base_path %}

<h1>Blog</h1>

{% for post in site.posts reversed %}
  {% unless post.date > site.time %}
    <div style="margin-bottom: 1.5em;">
      <h2 style="margin-bottom: 0.2em;">
        <a href="{{ base_path }}{{ post.url }}">{{ post.title }}</a>
      </h2>
      <p style="color: #888; font-size: 0.85em; margin-bottom: 0.3em;">{{ post.date | date: "%B %d, %Y" }}</p>
      {% if post.excerpt %}
        <p>{{ post.excerpt | strip_html | truncate: 200 }}</p>
      {% endif %}
    </div>
  {% endunless %}
{% endfor %}
