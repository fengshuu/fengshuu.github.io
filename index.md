---
layout: home
---

<div class="index-content blog">
    <div class="section">
        <ul class="artical-cate">
            <li class="on"><a href="/"><span>Blog</span></a></li>
            <li style="text-align:center"><a href="/tech"><span>Tech</span></a></li>
            <li style="text-align:right"><a href="/tag"><span>Tag</span></a></li>
        </ul>

        <div class="cate-bar"><span id="cateBar"></span></div>

        <ul class="artical-list">
        {% for post in site.categories.blog %}
            <li>
                <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
                <div class="title-desc">{{ post.description }}</div>
            </li>
        {% endfor %}
        </ul>
        {% for post in paginator.posts %}
            <a href="{{ post.url }}">{{ post.title }}</a>
        {% endfor %}

{% if paginator.previous_page %}
    <a href="{{ paginator.previous_page_path | prepend: site.baseurl | replace: '//', '/' }}"上一页</a>
{% endif %}
    
    </div>
    <div class="aside">
    </div>
</div>
