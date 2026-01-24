---
title: "Archives"
layout: splash  
permalink: /archives/
header:
  overlay_color: "#bbb" 
feature_row:
  - title: "<a href='/categories/'>📂 카테고리별 보기</a>"
  - title: "<a href='/tags/'>🏷️ 태그별 보기</a>"
  - title: "<a href='/year-archive/'>🗓️ 연도별 보기</a>"
--- 

{% include feature_row %}

<h2 style="text-align: center; margin-top: 50px;">Posts</h2>

<div class="entries-grid">
  {% for post in site.posts %}
    {% include archive-single.html type="grid" %}
  {% endfor %}
</div>