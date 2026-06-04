---
layout: default
title: Home
---

# Kagenti Wiki Research

A multi-agent research knowledge base.

## Pages

{% assign pages = site.pages | where_exp: "p", "p.path contains 'ai/'" | sort: "title" %}
{% for p in pages %}
{% unless p.path contains '_drafts' or p.name == 'index.md' or p.title == nil %}
- [{{ p.title | default: p.name }}]({{ p.url | relative_url }}){% if p.tags %} <span class="tags">{% for t in p.tags %}<span class="tag">{{ t }}</span>{% endfor %}</span>{% endif %}
{% endunless %}
{% endfor %}
