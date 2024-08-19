---
layout: non-post-page
title: Resources
permalink: resources/
---

Both for my own reference, and for easy sharing, here are links to resources ive found useful.


{% for category in site.data.resources.catagories %}

<h2> {{ category.display_title }} </h2>

<ul>
{% for resource in site.data.resources.resources %}

    {% if category.tag == resource.category %}
    <li>
        <a href="{{resource.url}}">{{resource.title}} </a>
    </li>
    {% endif %}

{% endfor %}
</ul>

{% endfor %}


