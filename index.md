---
title: GenAIOps 练习
permalink: index.html
layout: home
---

# 将生成式 AI 应用程序投入实际运行

以下快速入门练习旨在为你提供动手学习体验，帮助你探索在 Microsoft Azure 上将生成式 AI 工作负载投入实际运行所需的常见任务。

> **备注**：要完成这些练习，需要一个 Azure 订阅，其中包含足够的权限和配额以预配必要的 Azure 资源和生成式 AI 模型。 如果还没有 Azure 订阅，请注册 [Azure 帐户](https://azure.microsoft.com/free)。 针对新用户提供免费试用选项，其中包括前 30 天的积分。

## 快速入门练习

{% assign labs = site.pages | where_exp:"page", "page.url contains '/Instructions'" %} {% for activity in labs  %}
<hr>
### [{{ activity.lab.title }}]({{ site.github.url }}{{ activity.url }})

{{activity.lab.description}}

{% endfor %}

> **备注**：虽然可以自行完成这些练习，但它们旨在补充有关 [Microsoft Learn](https://learn.microsoft.com/training/paths/operationalize-gen-ai-apps/) 的模块；在其中可更深入地了解这些练习所基于的一些基础概念。
