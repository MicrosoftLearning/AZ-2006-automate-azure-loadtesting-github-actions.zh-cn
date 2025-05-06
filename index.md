---
title: 使用 GitHub Actions 自动执行 Azure 负载测试
permalink: index.html
layout: home
---

以下练习旨在为你提供实现 GitHub 操作和工作流以便使用 Azure 负载测试自动执行负载测试的动手学习体验。 

## 练习
<hr/>


{% assign labs = site.pages | where_exp:"page", "page.url contains '/Instructions/Labs'" %} {% for activity in labs  %}
* [{{ activity.lab.title }}]({{ site.github.url }}{{ activity.url }}) <br/> {{ activity.lab.description }} {% endfor %}
