---
layout: post
title: "How to Write a Jekyll Post"
date: 2025-07-25 10:00:00 -0000
author: "TimDude"
categories:
  - blog
tags:
  - blog
  - jekyll
---
## 🗓️ Filename

Create your post inside the `_posts/` folder.

Use this format: YYYY-MM-DD-title-with-dashes.md

This sets the **post's date** and **URL slug** automatically.

---
## 🧾 Front Matter

Each post needs this at the top:

~~~
---
title: "Your Post Title"
date: YYYY-MM-DD
layout: post
description: "Optional. Useful for SEO and feeds."
tags: [optional, list, of, tags]
---
~~~

## 🖼️ Where to Put Pictures

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

