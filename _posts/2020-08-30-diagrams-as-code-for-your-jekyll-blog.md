---
layout: post
title:  "Diagrams as code for your Jekyll blog"
date:   2020-08-30 00:00:00 -0500
categories: programming
---

[Mermaid](https://mermaid-js.github.io/mermaid/#/) is a Javascript library that allows you to define diagrams as code and it will render them for you on the page. It has support for common types of diagrams one would create when writing about software such as sequence diagrams, flow charts, ER diagrams etc ...

I tried integrating it into my Jekyll blog by using some of the existing plugins but didn't have much luck getting it to work reliably. I went back to basics and figured out the simplest way to include it.

Simply add this snippet to your `default.html` layout page, after the `</body>` tag. This will conditionally load the Mermaid JS and initialize it if you mark a specific post to require it.

```
{% raw %}
{% if page.mermaid %}
<script src="https://cdn.jsdelivr.net/npm/mermaid/dist/mermaid.min.js"></script>
<script>mermaid.initialize({startOnLoad:true});</script>
{% endif %}
{% endraw %}
```


In your individual post's Markdown you can then specify `markdown: true` as one of the headers.

To create individual diagrams you can write them as HTML like so:

```
<div class="mermaid">
sequenceDiagram
    participant P1
    participant P2
    P1->>P2: Hello world
</div>
```

And that's all it takes. Simpler than installing plugins, the JS will conditionally load only for pages that require it so you don't need to bog down the page load for posts that don't have diagrams. In my example, you are taking a dependency on a CDN to render your diagrams but you could also choose the self-host the Mermaid JS yourself.
