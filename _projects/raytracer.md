---
layout: default
---
# Raytracer - Masters Degree Class Project

<img src="/assets/images/HW_6_Fixed/basic-reflection.png" width="400"  display="inline-block"/>

During my masters degree at Drexel I took CS 636 - Advanced Rendering Techniques with Dr. David E Breen. This course was a continuation of the computer graphics course work at Drexel and was largely centered around an evolving CPU side raytracer. Starting from first algorithmic principles, I (and my fellow students) individually developed a program that created raytraced images and wrote about them. The articles below were written as milestone checkpoints through the quarter long class, each a response to a specific set of requirements assigned during the week or two given. They were originally published to a different site on gitlab, however I have moved them here, and relocated the corresponding source code to github as part of a portfolio unification.

I did a few things that went beyond the assigned milestones. One of the things that turned out to be the most useful for speedy development was pulling all of the code that hardcoded the location of objects in the scene out, and made the whole scene descritption driven by json files that could be fed in at run time. Meaning the same executable could be used to create multiple scenes. This isn't revolutionary when compared to mature game engines, but it was powerful in the context of this class. Discussion of this feature is in the Diffuse and Specular Shading With Point Lights article.

The source is on github now:
[Github repository link](https://github.com/TaylorEllington/CS636-CPU-Raytracer)

<h2>Posts about this project </h2>
<ul>
{% for post in site.tags.cs636-raytracer reversed %}
<li class="post-list-item">
    <span class="home-date">
      {{ post.date | date: site.theme_config.date_format }}»
    </span>
    <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
</li>
{%endfor%}
</ul>

