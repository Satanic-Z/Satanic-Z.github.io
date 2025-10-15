---
layout: null
permalink: /
---

<!doctype html>
<html lang="zh-CN">
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>{{ site.title | default: "Blog" }}</title>
<style>
  :root{
    --bg:#0b1020; --fg:#e6eefc; --muted:#9fb3d9; --card:#101735; --line:#1e2a52;
    --link:#a9c7ff; --link-hover:#c9dcff; --accent:#6ea8fe;
  }
  *{box-sizing:border-box}
  html,body{height:100%}
  body{
    margin:0; background:radial-gradient(1200px 600px at 10% -10%, #142046 0, transparent 50%),
               radial-gradient(1000px 600px at 110% 10%, #172452 0, transparent 50%),
               var(--bg);
    color:var(--fg); font:16px/1.7 system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial, sans-serif;
  }
  a{color:var(--link); text-decoration:none}
  a:hover{text-decoration:underline; color:var(--link-hover)}
  .wrap{
    max-width:820px; margin:8vh auto; padding:0 22px;
    display:flex; flex-direction:column; align-items:center; /* 中心对齐 */
  }
  header{ text-align:center; margin-bottom:28px }
  h1{ margin:0 0 6px; font-size:42px; letter-spacing:.3px }
  .sub{ color:var(--muted); margin:0; font-size:15px }
  .list{
    width:100%;
    display:flex; flex-direction:column; gap:14px;
    margin-top:22px;
  }
  .card{
    background:linear-gradient(180deg, rgba(255,255,255,.02), rgba(255,255,255,.005));
    border:1px solid var(--line); border-radius:18px; padding:18px 18px;
    transition:transform .12s ease, box-shadow .12s ease, border-color .12s ease;
  }
  .card:hover{
    transform:translateY(-1px);
    box-shadow:0 8px 22px rgba(0,0,0,.25);
    border-color:#2b3b75;
  }
  .title{
    display:block; font-size:20px; font-weight:700; margin:2px 0 6px;
  }
  .meta{
    font-size:13px; color:var(--muted); margin:0 0 10px;
  }
  .excerpt{
    margin:0; color:#d7e3ff; opacity:.92
  }
  footer{
    margin:34px 0 10px; color:var(--muted); font-size:12px; text-align:center;
  }
  /* 移动端微调 */
  @media (max-width:520px){
    h1{font-size:32px}
    .card{padding:16px}
  }
</style>

<body>
  <div class="wrap">
    <header>
      <h1>{{ site.title | default: "Yifan's Blog" }}</h1>
      <p class="sub">{{ site.description | default: "研究 / 随笔" }}</p>
    </header>

    <section class="list">
      {%- assign posts = site.posts -%}
      {%- if posts and posts != empty -%}
        {%- for post in posts limit: 20 -%}
          <article class="card">
            <a class="title" href="{{ post.url | relative_url }}">{{ post.title }}</a>
            <p class="meta">
              <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%Y-%m-%d" }}</time>
              {%- if post.tags and post.tags != empty -%} · {{ post.tags | join: " · " }}{%- endif -%}
            </p>
            <p class="excerpt">
              {{ post.excerpt | strip_html | normalize_whitespace | truncate: 140 }}
            </p>
          </article>
        {%- endfor -%}
      {%- else -%}
        <article class="card">
          <strong>还没有文章</strong>
          <p class="excerpt">在 <code>_posts/</code> 里新建一篇：<code>YYYY-MM-DD-title.md</code> 就会出现在这里。</p>
        </article>
      {%- endif -%}
    </section>

    <footer>© {{ "now" | date: "%Y" }} {{ site.author | default: "Yifan" }}</footer>
  </div>
</body>
</html>
