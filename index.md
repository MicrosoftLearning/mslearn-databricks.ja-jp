---
title: オンラインでホストされる手順
permalink: index.html
layout: home
---

# Azure Databricks の演習

これらの演習は、Microsoft Learn の次のトレーニング コンテンツをサポートするように設計されています。

- [Azure Databricks を使用してデータ レイクハウス分析ソリューションを実装する](https://learn.microsoft.com/training/paths/data-engineer-azure-databricks/)
- [Azure Databricks を使用して Machine Learning ソリューションを実装する](https://learn.microsoft.com/training/paths/build-operate-machine-learning-solutions-azure-databricks/)
- [Azure Databricks を使用してData Engineering ソリューションを実装する](https://learn.microsoft.com/training/paths/azure-databricks-data-engineer/)
- [Azure Databricks を使用して生成 AI エンジニアリングを実装する](https://learn.microsoft.com/training/paths/implement-generative-ai-engineering-azure-databricks/)

これらの演習を完了するには、管理者アクセス権が与えられている Azure サブスクリプションが必要です。

{% assign exercises = site.pages | where_exp:"page", "page.url contains '/Instructions'" %} {% for activity in exercises  %}
- [{{ activity.lab.title }}]({{ site.github.url }}{{ activity.url }}) | {% endfor %}
