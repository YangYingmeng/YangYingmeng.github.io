---
layout: page
title: 脚踏代码浪, 敲遍古今愁
description: 打码改变世界
keywords: Yingmeng Yang, 杨应猛
comments: true
menu: 关于
permalink: /about/
---

我是杨应猛，誓要脚踏代码浪, 敲遍古今愁

坚信「有道无术, 术尚可求」。

不断地学习 分享

## 联系

<ul>
{% for website in site.data.social %}
<li>{{website.sitename }}：<a href="{{ website.url }}" target="_blank">@{{ website.name }}</a></li>
{% endfor %}
{% if site.url contains 'mazhuang.org' %}
<li>
微信：<br />
<img style="height:192px;width:192px;border:1px solid lightgrey;" src="{{ site.url }}/assets/images/qrcode.jpg" alt="杨小猛" />
</li>
{% endif %}
</ul>


## Skill Keywords

{% for skill in site.data.skills %}
### {{ skill.name }}
<div class="btn-inline">
{% for keyword in skill.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
