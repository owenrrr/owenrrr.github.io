---
layout: default
title: tags
permalink: /tags/
---
<html>
    <style>
        .li-expo-posts {
            display: inline-block;
            width: 200px;
            text-overflow: ellipsis;
            white-space: nowrap;
            overflow: hidden;
        }
    </style>
<body>
    <div class="tags-expo">
        <div class="tags-expo-section">
            {% for tag in site.tags %}
            <h2 id="{{ tag[0] | slugify }}">{{ tag[0] }}</h2>
            <ul class="tags-expo-posts">
                {% for post in tag[1] %}
                <a href="{{ site.baseurl }}{{ post.url }}" class="li-expo-posts">
                    <li>
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
