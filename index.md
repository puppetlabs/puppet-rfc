---
layout: page
title: Armatures
---

# Armatures #

Here are all the current armatures:

<table>
    <th>
      <td>#</td><td>Title</td><td>Revision</td><td>Champion</td><td>Project</td>
    </th>
    {% for page in site.pages %}
      {% if page.main-page %}
        {% if page.arm < 10 %}
      <tr>
        <td>{{page.arm}}</td><td><a href="{{page.url}}">{{page.title}}</a></td><td>{{page.revision}}</td><td>{{page.champion}}</td><td>{{page.project}}</td>
      </tr>
        {% endif %}
      {% endif %}
    {% endfor %}
    {% for page in site.pages %}
      {% if page.main-page %}
        {% if page.arm >= 10 %}
      <tr>
        <td>{{page.arm}}</td><td><a href="{{page.url}}">{{page.title}}</a></td><td>{{page.revision}}</td><td>{{page.champion}}</td><td>{{page.project}}</td>
      </tr>
        {% endif %}
      {% endif %}
    {% endfor %}
</table>

