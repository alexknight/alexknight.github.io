---
title: 目录
layout: post
permalink: /articles/
dskey: laobie.github.io-articles.html
favicon: /assets/img/fav/articles.png
show_ad: false
---

{% for category in site.categories %}
<div class="divider"></div>
<div class="category-item">
    <div class="category-title" style="color:black; font-weight:bold; font-size: 125%">
        {{ category | first | capitalize }} - 共{{ category | last | size }}篇
    </div>
    <!-- <div class="category-count">{{ category | last | size }}</div> -->
</div>

{% for post in category.last %}
<div class="article-item">
    <div class="article-title">
        <a href="{{ post.url }}" target="_blank">{{ post.title }}</a>
        <span class="article-date">{{post.date | date:"%Y %m-%d"}}</span>
    </div>
</div>

{% endfor %}
<br />

{% endfor %}
