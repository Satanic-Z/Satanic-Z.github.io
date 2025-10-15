---
layout: null
---
<!doctype html>
<html lang="zh-CN">
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>{{ page.title }} · {{ site.title }}</title>
<style>
  body{margin:0;background:#0b1020;color:#e6eefc;font:16px/1.7 system-ui,-apple-system,Segoe UI,Roboto,Helvetica,Arial,sans-serif}
  .wrap{max-width:820px;margin:8vh auto;padding:0 22px}
  a{color:#a9c7ff} a:hover{text-decoration:underline}
  h1{font-size:36px;margin:.2em 0 .2em}
  .meta{color:#9fb3d9;font-size:13px;margin-bottom:18px}
  article{background:#101735;border:1px solid #1e2a52;border-radius:18px;padding:22px}
  article img{max-width:100%}
  pre{overflow:auto;background:#0f1630;padding:14px;border-radius:12px;border:1px solid #1e2a52}
  code{font-family:ui-monospace,SFMono-Regular,Menlo,Consolas,monospace}
  footer{color:#9fb3d9;font-size:12px;text-align:center;margin:28px 0}
</style>
<body>
  <div class="wrap">
    <h1>{{ page.title }}</h1>
    <p class="meta"><time datetime="{{ page.date | date_to_xmlschema }}">{{ page.date | date: "%Y-%m-%d" }}</time>{% if page.tags %} · {{ page.tags | join: " · " }}{% endif %}</p>
    <article>{{ content }}</article>
    <footer><a href="{{ "/" | relative_url }}">← 返回首页</a></footer>
  </div>
</body>
</html>
