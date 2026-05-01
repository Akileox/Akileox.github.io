---
layout: default
permalink: /blog/
author_profile: false
classes: wide
---

<div class="blog-wrap">

  <h1 style="font-size:2rem;font-weight:800;color:var(--text);margin:0 0 0.5rem;letter-spacing:-0.02em;">Blog</h1>
  <p style="color:var(--text-muted);font-size:0.95rem;margin:0 0 2rem;">To share my ideas and experiences, and what I'm working on.</p>

  <!-- Category Filter -->
  <div class="blog-filter">
    <button class="blog-filter__btn active" data-filter="all">All</button>
    <button class="blog-filter__btn" data-filter="Daily">Daily</button>
    <button class="blog-filter__btn" data-filter="Idea">Idea</button>
    <button class="blog-filter__btn" data-filter="Project">Project</button>
    <button class="blog-filter__btn" data-filter="Mentoring">Mentoring</button>
  </div>

  <!-- Post List -->
  <div class="blog-list" id="blog-list">
    {% for post in site.posts %}
    <a href="{{ post.url | relative_url }}"
       class="blog-list-item"
       data-category="{{ post.categories.first }}">
      <div class="blog-list-item__img">
        {% if post.header.teaser %}
          <img src="{{ post.header.teaser | relative_url }}" alt="{{ post.title }}">
        {% endif %}
      </div>
      <div class="blog-list-item__content">
        {% assign pcat = post.categories.first | downcase %}
        <span class="cat-badge cat-badge--{{ pcat }}">{{ post.categories.first | default: "Post" }}</span>
        <h2 class="blog-list-item__title">{{ post.title }}</h2>
        <p class="blog-list-item__excerpt">{{ post.excerpt | strip_html | truncatewords: 30 }}</p>
        <p class="blog-list-item__meta">{{ post.date | date: "%Y. %m. %d" }}</p>
      </div>
    </a>
    {% endfor %}
  </div>

</div>

<script>
  const btns = document.querySelectorAll('.blog-filter__btn');
  const items = document.querySelectorAll('.blog-list-item');

  btns.forEach(btn => {
    btn.addEventListener('click', () => {
      btns.forEach(b => b.classList.remove('active'));
      btn.classList.add('active');

      const filter = btn.dataset.filter;
      items.forEach(item => {
        if (filter === 'all' || item.dataset.category === filter) {
          item.style.display = 'flex';
        } else {
          item.style.display = 'none';
        }
      });
    });
  });
</script>
