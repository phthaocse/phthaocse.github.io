---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults




layout: default 

---
<!-- 
<div id="archives">
  {% for category in site.categories %}
    <div class="archive-group">
      {% capture category_name %}{{ category | first }}{% endcapture %}
      <div id="#{{ category_name | slugize }}"></div>
      <h1 class="category-head">{{ category_name }}</h1>
      <a name="{{ category_name | slugize }}"></a>
      {%- assign date_format = site.minima.date_format | default: "%b %-d, %Y" -%}

      {% for post in site.categories[category_name] %}
      <article class="archive-item">
        <span class="post-meta">{{ post.date | date: date_format }}</span>
        <h3>
          <a class="post-link" href="{{ post.url | relative_url }}">
            {{ post.title | escape }}
          </a>
        </h3>
        {%- if site.show_excerpts -%}
          {{ post.excerpt }}
        {%- endif -%}
      </article>
      {% endfor %}
    </div>
  {% endfor %}
</div>  -->
<div class="home">
  {%- if page.title -%}
    <h1 class="page-heading">{{ page.title }}</h1>
  {%- endif -%}



  {% if site.paginate %}
    {% assign posts = paginator.posts %}
  {% else %}
    {% assign posts = site.posts %}
  {% endif %}


  {%- if posts.size > 0 -%}
    {%- if page.list_title -%}
      <h2 class="post-list-heading">{{ page.list_title }}</h2>
    {%- endif -%}
<div id="archives">
  {% for category in site.categories %}
    
<div class="archive-group">
      {% capture category_name %}{{ category | first }}{% endcapture %}
      <div id="#{{ category_name | slugize }}"></div>
      <h1 class="category-head">{{ category_name }}</h1>
      <a name="{{ category_name | slugize }}"></a>

      {%- assign date_format = site.minima.date_format | default: "%b %-d, %Y" -%}
    <ul class="post-list">
      {%- for post in site.categories[category_name] -%}
      <li>
        <span class="post-meta">{{ post.date | date: date_format }}</span>
        <h3>
          <a class="post-link" href="{{ post.url | relative_url }}">
            {{ post.title | escape }}
          </a>
        </h3>
        {%- if site.show_excerpts -%}
          {{ post.excerpt }}
        {%- endif -%}
      </li>
      {%- endfor -%}
    </ul>
{% endfor %}

    

    {% if site.paginate %}
      <div class="pager">
        <ul class="pagination">
        {%- if paginator.previous_page %}
          <li><a href="{{ paginator.previous_page_path | relative_url }}" class="previous-page">{{ paginator.previous_page }}</a></li>
        {%- else %}
          <li><div class="pager-edge">•</div></li>
        {%- endif %}
          <li><div class="current-page">{{ paginator.page }}</div></li>
        {%- if paginator.next_page %}
          <li><a href="{{ paginator.next_page_path | relative_url }}" class="next-page">{{ paginator.next_page }}</a></li>
        {%- else %}
          <li><div class="pager-edge">•</div></li>
        {%- endif %}
        </ul>
      </div>
    {%- endif %}

  {%- endif -%}

</div>