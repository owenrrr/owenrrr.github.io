---
layout: default
title: tags
permalink: /tags/
---
<html>
<body>
    <div class="tags-expo">
        <div class="tags-expo-section">
            {% for tag in site.tags %}
            <h2 id="{{ tag[0] | slugify }}">{{ tag[0] }}</h2>
            <ul class="tags-expo-posts">
                {% for post in tag[1] %}
                <a href="{{ site.baseurl }}{{ post.url }}" style="display: inline-block; width: 33%;">
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
