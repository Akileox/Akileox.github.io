---
layout: default
permalink: /blog/
author_profile: false
classes: wide
---

<div markdown="0"><a href="/requests/" class="blog-request-fab" title="요청함 — 논문 분석·아이디어 요청" aria-label="요청함"><svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M21 15a2 2 0 0 1-2 2H7l-4 4V5a2 2 0 0 1 2-2h14a2 2 0 0 1 2 2z"/></svg><span>요청함</span></a></div>

<div class="blog-wrap">

  <h1 style="font-family:var(--font-serif);font-size:2rem;font-weight:700;color:var(--text);margin:0 0 0.5rem;letter-spacing:-0.01em;">Blog</h1>
  <p style="color:var(--text-muted);font-size:0.95rem;margin:0 0 2rem;">To share my ideas and experiences, and what I'm working on.</p>

  <!-- Category Filter -->
  <div class="blog-filter">
    <button class="blog-filter__btn active" data-filter="all">All</button>
    <button class="blog-filter__btn" data-filter="Daily">Daily</button>
    <button class="blog-filter__btn" data-filter="Idea">Idea</button>
    <button class="blog-filter__btn" data-filter="Project">Project</button>
    <button class="blog-filter__btn" data-filter="Mentoring">Mentoring</button>
    <button class="blog-filter__btn" data-filter="AI">AI</button>
  </div>

  <!-- Post List -->
  <div class="blog-list" id="blog-list">
    {% for post in site.posts %}
    <a href="{{ post.url | relative_url }}"
       class="blog-list-item"
       data-category="{{ post.categories.first }}">
      {% if post.header.teaser %}
      <div class="blog-list-item__img">
        <img src="{{ post.header.teaser | relative_url }}" alt="{{ post.title }}">
      </div>
      {% endif %}
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
