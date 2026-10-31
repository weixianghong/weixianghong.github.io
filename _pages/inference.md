---
layout: archive
title: "LLM 学习笔记"
permalink: /blogs/inference/
author_profile: true
lang: zh-CN
extra_css: /assets/css/inference-series.css
excerpt: "学到哪里 写到哪里 未完待续 敬请指教"
---

{% include base_path %}

<header class="series-heading">
{% assign series_minutes = site.posts | where: 'series', 'inference' | map: 'read_time_minutes' | sort %}
<span class="series-kicker">5 篇中文笔记 · 每篇约 {{ series_minutes.first }}–{{ series_minutes.last }} 分钟</span>
<h2>{{ site.data.inference_series.title }}</h2>
<p>{{ site.data.inference_series.subtitle | escape | newline_to_br }}</p>
</header>

<ol class="series-chapters">
{% for chapter in site.data.inference_series.chapters %}
{% assign chapter_post = site.posts | where: "url", chapter.url | first %}
<li><a class="series-chapter" href="{{ base_path }}{{ chapter.url }}"><span class="series-chapter__number" aria-hidden="true">{{ chapter.number }}</span><span><span class="series-chapter__title">{{ chapter.title }}</span><span class="series-chapter__deck">{{ chapter.subtitle }}</span><span class="series-chapter__meta">{{ chapter.topic }} &nbsp;·&nbsp; {{ chapter_post.read_time_minutes }} 分钟</span></span><span class="series-chapter__arrow" aria-hidden="true">↗</span></a></li>
{% endfor %}
</ol>
