---
title: 目录
layout: post
permalink: /articles/
dskey: laobie.github.io-articles.html
favicon: /assets/img/fav/articles.png
show_ad: false
---

{% for category in site.categories %}
<div class="category-item">
    <div class="blogentry">
        <div class="category-title" >{{ category | first | capitalize }}({共{ category | last | size }}篇)</div>
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
{% endfor %}

.demo_line_01 {
    padding: 0 20px 0;
    margin: 20px 0;
    line-height: 1px;
    border-left: 200px solid #ddd;
    border-right: 200px solid #ddd;
    text-align: center;
}
