---
layout: archive
title: "Blog"
permalink: /blog/
author_profile: true
---

{% include base_path %}

{% for post in site.posts %}
  <div style="margin-bottom: 2.5em; padding-bottom: 1.5em; border-bottom: 1px solid #ddd;">
    <h2 style="margin-bottom: 0.1em; margin-top: 0;">
      <a href="{{ base_path }}{{ post.url }}" style="text-decoration: none;">{{ post.title }}</a>
    </h2>
    <p style="color: #888; font-size: 0.85em; margin-bottom: 0.5em;">
      {{ post.date | date: "%B %d, %Y" }}
      &middot;
      {% capture words %}{{ post.content | number_of_words }}{% endcapture %}
      {% if words contains "-" %}1{% else %}{{ words | plus: 250 | divided_by: 250 }}{% endif %} min read
    </p>
    {% if post.tags.size > 0 %}
    <p style="margin-bottom: 0.5em;">
      {% for tag in post.tags %}
        <span style="display: inline-block; background: #f0f0f0; color: #555; padding: 2px 8px; border-radius: 3px; font-size: 0.75em; margin-right: 4px;">{{ tag }}</span>
      {% endfor %}
    </p>
    {% endif %}
    {% if post.excerpt %}
      <p style="color: #555;">{{ post.excerpt | strip_html | truncate: 250 }}</p>
    {% endif %}
  </div>
{% endfor %}
