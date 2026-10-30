---
layout: archive
title: "LLM 学习笔记"
permalink: /blogs/inference/
author_profile: true
lang: zh-CN
extra_css: /assets/css/inference-series.css
excerpt: "我的大模型学习笔记，从 QKV 一路挖到显存、推理芯片和 API 账单。"
---

{% include base_path %}

<header class="series-heading">
<span class="series-kicker">5 篇中文笔记 · 每篇约 6–8 分钟</span>
<h2>{{ site.data.inference_series.title }}</h2>
<p>{{ site.data.inference_series.subtitle }}</p>
</header>

<p class="series-intro">最初只是想弄明白 QKV 怎么来的，结果一路挖到了显存、晶圆和 API 账单。坑越挖越大，先把目前想明白的记下来，免得过两个月又得从头查。还没弃坑，笔记先放这儿。</p>

<ol class="series-chapters">
{% for chapter in site.data.inference_series.chapters %}
<li><a class="series-chapter" href="{{ base_path }}{{ chapter.url }}"><span class="series-chapter__number" aria-hidden="true">{{ chapter.number }}</span><span><span class="series-chapter__title">{{ chapter.title }}</span><span class="series-chapter__deck">{{ chapter.subtitle }}</span><span class="series-chapter__meta">{{ chapter.topic }} &nbsp;·&nbsp; {{ chapter.minutes }} 分钟</span></span><span class="series-chapter__arrow" aria-hidden="true">↗</span></a></li>
{% endfor %}
</ol>
