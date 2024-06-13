---
layout: default
title: Tags
permalink: /tags/
---
<html>
<style>
    .li-expo-posts {
        white-space: nowrap;
        text-overflow: ellipsis;
        width: 200px;
        display: inline-block;
        overflow: hidden
    }
</style>
<body>
    <div class="tags-expo">
        <div class="tags-expo-section">
            {% for tag in site.tags %}
            <h2 id="{{ tag[0] | slugify }}">{{ tag[0] }}</h2>
            <ul class="tags-expo-posts">
                {% for post in tag[1] %}
                <a href="{{ site.baseurl }}{{ post.url }}">
                    <li class="li-expo-posts">
                        {{ post.title }}
                    </li>
                </a>
                {% endfor %}
            </ul>
            {% endfor %}
        </div>
    </div>
</body>

</html>
