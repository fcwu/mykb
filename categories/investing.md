---
title: 投資理財
description: 投資理財外部教學素材摘要與批判（CLEC 系列、指數化投資觀念）
permalink: /categories/investing.html
---

# 投資理財

> 投資理財外部教學素材摘要與批判（CLEC 系列、指數化投資觀念）。摘要忠實呈現原意，**不代表認可任何具體投資/財務行動**。

{% assign in_cat = site.pages | where: "category", "investing" | sort: "date" | reverse %}
{% for p in in_cat %}
- **[{{ p.title }}]({{ p.url | relative_url }})** <small>· {{ p.date | date: "%Y-%m-%d" }}</small>
  {% if p.description %}<br><small>{{ p.description }}</small>{% endif %}
{% endfor %}

---

<small>共 {{ in_cat | size }} 篇 · [回首頁](/) · [📡 RSS](/feed.xml)</small>
