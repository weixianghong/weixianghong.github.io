---
layout: archive
permalink: /blog/
author_profile: true
---

{% include base_path %}

{% capture written_year %}'None'{% endcapture %}
{% for post in site.posts %}
{% capture year %}{{ post.date | date: '%Y' }}{% endcapture %}
{% if year != written_year %}
<h2 class="group-title">{{ year }}</h2>
{% capture written_year %}{{ year }}{% endcapture %}
{% endif %}
<div class="post-card">
<h2 class="post-card__title"><a href="{{ base_path }}{{ post.url }}">{{ post.title }}</a></h2>
<p class="post-card__meta">
{{ post.date | date: "%B %d, %Y" }}
&middot;
{% assign words = post.content | strip_html | number_of_words %}
{% if words < 180 %}less than 1 minute read{% elsif words < 360 %}1 minute read{% else %}{{ words | divided_by: site.words_per_minute }} minute read{% endif %}
</p>
{% if post.tags.size > 0 %}
<p class="post-card__tags">{% for tag in post.tags %}<span class="post-tag">{{ tag }}</span>{% endfor %}</p>
{% endif %}
{% if post.excerpt %}
<p class="post-card__excerpt">{{ post.excerpt | strip_html | truncate: 250 }}</p>
{% endif %}
</div>
{% endfor %}
