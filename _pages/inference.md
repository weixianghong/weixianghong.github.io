---
layout: archive
title: "推理实战"
permalink: /blogs/inference/
author_profile: true
lang: zh-CN
extra_css: /assets/css/inference-series.css
excerpt: "五篇中文笔记：从 QKV、显存带宽，讲到 Cerebras、Groq，以及 KV Cache 和 Agent 的上下文管理。"
---

{% include base_path %}

<header class="series-heading">
<span class="series-kicker">5 篇中文笔记 · 每篇约 6–8 分钟</span>
<h2>{{ site.data.inference_series.title }}</h2>
<p>{{ site.data.inference_series.subtitle }}</p>
</header>

<p class="series-intro">模型的公式看懂了，为什么真跑起来还是慢？这个系列沿着数据走一遍：先看 Attention 怎么读信息，再算 GPU 搬参数的账，接着看两种推理芯片怎么换路走，最后回到每次 API 调用都碰得到的 KV Cache。可以从头读，也可以直接挑关心的一篇。</p>

<ol class="series-chapters">
{% for chapter in site.data.inference_series.chapters %}
<li><a class="series-chapter" href="{{ base_path }}{{ chapter.url }}"><span class="series-chapter__number" aria-hidden="true">{{ chapter.number }}</span><span><span class="series-chapter__title">{{ chapter.title }}</span><span class="series-chapter__deck">{{ chapter.subtitle }}</span><span class="series-chapter__meta">{{ chapter.topic }} &nbsp;·&nbsp; {{ chapter.minutes }} 分钟</span></span><span class="series-chapter__arrow" aria-hidden="true">↗</span></a></li>
{% endfor %}
</ol>
