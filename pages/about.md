---
layout: page
title: 个人简介
description: 持续学习，乐于分享
keywords: 莫轩然, Moxuanran
comments: true
menu: 关于我
permalink: /about/
---

我是莫轩然，持续学习，拥抱变化。


## 联系

{% for website in site.data.social %}
* {{ website.sitename }}：[@{{ website.name }}]({{ website.url }})
{% endfor %}

## Skill Keywords

{% for category in site.data.skills %}
### {{ category.name }}
<div class="btn-inline">
{% for keyword in category.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
