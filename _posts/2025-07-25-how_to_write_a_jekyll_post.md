---
layout: post
title: "How to Write a Jekyll Post"
date: 2025-07-25 10:00:00 +0200
author: "TimDude"
categories: blog
tags: blog jekyll
---

Hello my name is Johnny Cash

## Filename

Create your post inside the `_posts/` folder.

Use this format: YYYY-MM-DD-title-with-dashes.md

This sets the **post's date** and **URL slug** automatically.

## Front Matter

Each post needs this at the top:

~~~
Be aware to use your time Zone or else your post might seems like its in the future and not get
published right away.
---
title: "Your Post Title"
date: YYYY-MM-DD 10:00:00 +0200
layout: post
description: "Optional. Useful for SEO and feeds."
tags: [optional, list, of, tags]
---
~~~

## Where to Put Pictures

~~~
assets/images/blog/YYYY-MM-DD-title/
~~~
example
~~~
assets/images/blog/2025-07-25-how-to-write-a-jekyll-post/header.jpg
~~~
reference like this:
~~~
![Header]({{ "/assets/images/blog/2025-07-25-how-to-write-a-jekyll-post/header.jpg" | relative_url }})
~~~
