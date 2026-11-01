---
layout: archive
title: "Blogs"
extra_css: /assets/css/inference-series.css
permalink: /blogs/
redirect_from:
  - /blog/
author_profile: true
---

{% include base_path %}

{% assign written_year = '' %}
{% assign listed_series_keys = '' %}
{% for post in site.posts %}
{% assign series_definition = site.data.blog_series[post.series] %}
{% if series_definition %}
  {% assign series_key = post.series | prepend: '|' | append: '|' %}
  {% if listed_series_keys contains series_key %}{% continue %}{% endif %}
  {% assign listed_series_keys = listed_series_keys | append: series_key %}
{% endif %}
{% assign year = post.date | date: '%Y' %}
{% if year != written_year %}
<h2 class="group-title">{{ year }}</h2>
{% assign written_year = year %}
{% endif %}
{% if series_definition %}
  {% include blog-series-card.html definition=series_definition date=post.date %}
{% else %}
<div class="post-card">
<h2 class="post-card__title"><a href="{{ base_path }}{{ post.url }}">{{ post.title }}</a></h2>
<p class="post-card__meta">
{{ post.date | date: "%B %d, %Y" }}
&middot;
{% if post.read_time_minutes %}{{ post.read_time_minutes }} minute read{% else %}{% assign words = post.content | strip_html | number_of_words %}{% if words < 180 %}less than 1 minute read{% elsif words < 360 %}1 minute read{% else %}{{ words | divided_by: site.words_per_minute }} minute read{% endif %}{% endif %}
</p>
{% if post.tags.size > 0 %}
<p class="post-card__tags">{% for tag in post.tags %}<span class="post-tag">{{ tag }}</span>{% endfor %}</p>
{% endif %}
{% if post.excerpt %}
<p class="post-card__excerpt">{{ post.excerpt | strip_html | truncate: 250 }}</p>
{% endif %}
</div>
{% endif %}
{% endfor %}
