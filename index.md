---
title: 联机托管说明
permalink: index.html
layout: home
---

# Microsoft Learn - 动手练习

以下动手练习旨在支持 [Microsoft Learn](https://docs.microsoft.com/training/) 培训。

{% assign labs = site.pages | where_exp:"page", "page.url contains '/Instructions'" %}
| |
| --- | --- | 
{% 表示实验室 % 中的活动}| [{{ activity.lab.title }}{% if activity.lab.type %} - {{ activity.lab.type }}{% endif %}]({{ site.github.url }}{{ activity.url }}) |
{% endfor %}
