---
title:  "A Quick Introduction"
date:   2015-03-09 22:00:25
tags: [jekyll, blog, meta]
---

Hello, and welcome to my blog! I'm not completely sure what I will use the blog
for, but I figure any time I solve some problem that other people may also run
in to, I should probably post about it.

## About the blog
After exploring a few different options for how to set up the blog, including
WordPress and Tumblr, I settled on [jekyll](http://jekyllrb.com). According
to them, "Jekyll is a simple, blog-aware, static site generator perfect
for personal, project, or organization sites", which sounds exactly like
what I need.

Some highlights of Jekyll that made me go with it are:

 - Pages are statically generated, so they can be placed anywhere on an existing
   site, without using any external services, frameworks, subdomains, etc.
 - Pages are created in Markdown! And with `jekyll serve`, changes in the
   markdown files are monitored and automatically refreshed.
 - Awesome support for syntax highlighting.

So I can type things like this<sup>[<a href="#foot1">1</a>]</sup>:

<div class="highlight">
<pre><code><span class="n">```</span><span class="k">scala</span>
<span class="n">object Main extends App {
  def greeting(name: String) = s"Hello $name!"
  println(greeting("Jekyll"))
}
```</span></code></pre>
</div>

And it comes out like this:

```scala
object Main extends App {
  def greeting(name: String) = s"Hello $name!"
  println(greeting("Jekyll"))
}
```

Even Scala REPL highlighting looks ok (although it could use some work):

```scala
scala> def greeting(name: String) = s"Hello $name!"
greeting: (name: String)String

scala> println(greeting("Jekyll"))
Hello Jekyll!
```

You can see the source for this post and the entire Jekyll project
on GitHub here
<a href="https://github.com/iantabolt/iantabolt.github.io"><button type="button" class="btn btn-default">View On GitHub</button></a>

-----------
<div id="foot1">
[1] GitHub flavored markdown triple-backtick
sytntax supported by using `markdown: redcarpet` in `_config.yml`.
</div>
