# README

Personal blog in English

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>

Checkout tech blog in Spanish at: [culpeoit.github.io](https://culpeoit.github.io)
