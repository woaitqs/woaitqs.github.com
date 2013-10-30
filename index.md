---
layout: default
title: Entrance for this blog.
---
{% include JB/setup %}

<div id="content">
    <div class="text-post posts">
        {% for post in site.posts limit:5 %}
                <h2><a class="post_title" href="{{ post.url }}">{{post.title}}</a></h2>
                <div class="caption rich-content">
                        {{ post.content }}
                </div>
        {% endfor %}
    </div>
</div>

