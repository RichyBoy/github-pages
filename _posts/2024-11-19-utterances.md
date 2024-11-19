---
title: "Comments section on github pages with Utterence"
date: 2024-11-19
layout: default
tags: jekyll blog github utterence minima
---

# Comments powered by Utterence

For better (or for worse!) I would like my blog to have a comments section. I selected Utterences as it seems pretty simple and I like the interface. However, it took me nearly a day to get it working with the Minima theme, which is the theme you use as default when going through the github-pages tutorial. To be fair most of this time was spent learning about Jekyll, which I knew nothing of at all and personally there is little I dislike more than web dev (yuck, yuckity yuck etc).

## Dive straight in

The repo for Utterances lives at [https://github.com/utterance/utterances](https://github.com/utterance/utterances) and the other page you will need is [https://utteranc.es/](https://utteranc.es/). However, mostly what you need to know: You don't need a repo in github.io and you don't need OAuth.

To re-iterate one of the links:

  1. Make sure the repo is public, otherwise your readers will not be able to view the issues/comments.
  2. Make sure the utterances app is installed on the repo, otherwise users will not be able to post comments. 

If you've never installed a GitHub app before, navigate to your repo, then Settings and then under Integrations is an icon for GitHub apps.

## What you need to do in your project

A typical Jekyll project will have content such as includes, layouts, posts etc all starting with a leading underscore. If you have completed the tutorial you will know about _posts. The minima theme hides a lot of this away, what you need to do is override the post.html layout so that the script code for utterences can be inserted. The HTML code for this lives at [https://github.com/jekyll/minima/blob/master/_layouts/post.html](https://github.com/jekyll/minima/blob/master/_layouts/post.html). You can probably just drop the script source straight in, however I decided to put it in an includes directory. This is what my project now looks like with two new html files, `comments.html` and `post.html` inside two new directories, `_includes` and `_layouts`.

![Image](/assets/images/project.png)

The source code for `comments.html` is below, and was created by the generator in one of the two links above:

```
<script src="https://utteranc.es/client.js" 
  repo="RichyBoy/github-pages" 
  issue-term="pathname" 
  label="scribblings"
  theme="github-light" 
  crossorigin="anonymous" async>
</script>
```

My overriden `post.html` looks like this:

```{% raw %}
---
layout: default
---
<article class="post h-entry" itemscope itemtype="http://schema.org/BlogPosting">

<header class="post-header">
    <h1 class="post-title p-name" itemprop="name headline">{{ page.title | escape }}</h1>
    <p class="post-meta">
    <time class="dt-published" datetime="{{ page.date | date_to_xmlschema }}" itemprop="datePublished">
        {%- assign date_format = site.minima.date_format | default: "%b %-d, %Y" -%}
        {{ page.date | date: date_format }}
    </time>
    {%- if page.author -%}
        • <span itemprop="author" itemscope itemtype="http://schema.org/Person"><span class="p-author h-card" itemprop="name">{{ page.author }}</span></span>
    {%- endif -%}</p>
</header>

<div class="post-content e-content" itemprop="articleBody">
    {{ content }}
</div>

shortname: fake_disqus_utterances
    {% include comments.html %}
    
<a class="u-url" href="{{ page.url | relative_url }}" hidden></a>
</article>{% endraw %}
```

The actual change is right at the bottom:
```{% raw %}
shortname: fake_disqus_utterances
  {% include comments.html %}{% endraw %}
```
        
from

```{% raw %}
{%- if site.disqus.shortname -%}
  {%- include disqus_comments.html -%}
{%- endif -%}{% endraw %}
```

Probably don't need the shortname, and probably could place the script code in directly, I also notice the git repository has 'base' in the frontmatter yet I have default.. maybe I grabbed an older version, any, all I know is that it works!

The only other change is to make sure your frontmatter is picking up the default override, for example on this very page:
```{% raw %}
---
title: "Comments section on github pages with Utterence"
date: 2024-11-19
layout: default
tags: jekyll blog github utterence minima
---{% endraw %}
```

instead of a layout of home etc. I am sure when more experienced with Jekyll such changes become more natural and understandable.

