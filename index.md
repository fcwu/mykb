---
title: Doro Digest
description: Doro 的個人筆記摘要與外部素材整理
---

## 最新

{% assign content_pages = site.pages | where_exp: "p", "p.category" | sort: "date" | reverse %}
{% for p in content_pages limit: 5 %}
- **[{{ p.title }}]({{ p.url | relative_url }})** <small>· {{ p.date | date: "%Y-%m-%d" }} · {{ p.category }}</small>
  {% if p.description %}<br><small>{{ p.description }}</small>{% endif %}
{% endfor %}

---

## 分類

- **[AI 與 Agent 工程](categories/ai-agent.html)** — Agent Memory、Computer-Use Agent、Harness Engineering
- **[投資理財](categories/investing.html)** — CLEC 系列、指數化投資觀念

---

<small>共 {{ content_pages | size }} 篇 · [完整 RSS feed](feed.xml)</small>
