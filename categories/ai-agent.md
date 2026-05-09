---
title: AI 與 Agent 工程
description: Agent 設計、記憶架構、Computer-Use Agent、Harness Engineering 等技術摘要
permalink: /categories/ai-agent.html
---

# AI 與 Agent 工程

> Agent 設計、記憶架構、Computer-Use Agent、Harness Engineering 等技術摘要。

{% assign in_cat = site.pages | where: "category", "ai-agent" | sort: "date" | reverse %}
{% for p in in_cat %}
- **[{{ p.title }}]({{ p.url | relative_url }})** <small>· {{ p.date | date: "%Y-%m-%d" }}</small>
  {% if p.description %}<br><small>{{ p.description }}</small>{% endif %}
{% endfor %}

---

<small>共 {{ in_cat | size }} 篇 · [回首頁](/) · [📡 RSS](/feed.xml)</small>
