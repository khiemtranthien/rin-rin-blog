---
layout: post
title:  "Get started with Jekyll!"
date:   2021-08-12 00:13:37 +0700
categories: jekyll-tutorial
tag: jekyll
---
# Launch

- New site: `sjekyll new myblog`
- Change to you blog folder: `cd myblog`
- Run local server: `bundle exec jekyll serve`

# Create post
- File format:

  `YEAR-MONTH-DAY-title.MARKUP` 

  For example: `2021-08-12-welcome-to-jekyll.markdown`  

- Highlight:
  - Simple highlight: `` `text here` ``
  - Code highlight:  
![Highlight code in post](/rin-rin-blog/assets/images/jekyll_tutorial/highlight_code.png){: width="300"}  
  Result:  
    {% highlight ruby %}
    def print_hi(name)
      puts "Hi, #{name}"
    end
    print_hi('Tom')
    #=> prints 'Hi, Tom' to STDOUT.
    {% endhighlight %}

- Display image  
    - /rin-rin-blog: base path configured in _config.yml  
    - /assets/images/jekyll_tutorial/highlight_code.png: image file in assets folder  
    - \{: width="300" height="400" \}: image style  

  {% highlight html %}
    ![Highlight code in post](/rin-rin-blog/assets/images/jekyll_tutorial/highlight_code.png){: width="300"}
  {% endhighlight %}  


# Organize posts
You can create any folder inside `_posts` like `jekyll-tutorial`, `block-chain`, etc to group your posts. It doesn't affect how Jekyll renders actual posts in those folders.
