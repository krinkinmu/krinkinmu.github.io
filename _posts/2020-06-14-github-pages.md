---
layout: default
title: GitHub Pages and Jekyll
excerpt_separator: <!--more-->
tags: github-pages jekyll
---

[GitHubPages]: https://guides.github.com/features/pages/ "Getting Started with GitHub Pages"
[Jekyll]: https://jekyllrb.com/ "Jekyll Home Page"
[JekyllPlugins]: https://jekyllrb.com/docs/plugins/ "Jekyll Plugins"
[JekyllStructure]: https://import.jekyllrb.com/ "Jekyll Directory Structure"
[Liquid]: https://shopify.github.io/liquid/ "Liquid Home Page"
[Markdown]: https://guides.github.com/features/mastering-markdown/ "Markdown Guide"
[FrontMatter]: https://jekyllrb.com/docs/front-matter/ "Jekyll Front Matter"
[JekyllVariables]: https://jekyllrb.com/docs/variables/ "Jekyll Varibales"

I'm mostly trying to stay away from any kind of front-end related engineering
because I've never found it interesting enough to spend a reasonable amount of
time to learn it.

That being said, on a subjective level I like the idea of keeping my posts in
VCS close to the code, so I started looking at [GitHub Pages][GitHubPages].
What follows is a very brief explanation of what I found with references to
other sources.

<!--more-->

# Getting Started

[Getting Started with GitHub Pages][GitHubPages] is a simple guide that explains
how to start. The basic idea is rather simple:

1. Create a GitHub repository for your documentation/blog/etc
2. Pick the theme that suits your prefernce
3. Commit the content you want to the repository in your prefered markup
   language
4. GitHub handles the rest

It works great for static content, like documentation with cross references for
easy navigation. It should also be enough to create a blog as well.

> **NOTE:** It doesn't mean that you cannot put any dynamic elements on GitHub
Pages. You can probably put any JavaScript you want and use it to talk to some
backend if you so desire, though I didn't personally try it. However GitHub
Pages will not host the backend.

However there are some features that you might want for a blog. For example, you
can encode the list of all your posts manually, but it would be nice if the list
of posts with links to them was generated automatically for you.

# Going a little bit deeper

One you decided to go deeper, if you, like myself, are completely oblivious of
the framework used under the hood you might get lost in a bunch of new terms.
Let's try to takcle new terms one at a time.

## Jekyll

GitHub Pages uses [Jekyll][Jekyll]. Jekyll in simple terms is a tool that from
a description generates a static site.

Why would we need this tool if we can just say create a bunch of interlinked
HTML pages?

Well, Jekyll allows for some additional processing that may help to automate
some manual tasks. Returning to the example below, Jekyll may automatically
discover the list of posts you have and more or less generate the index page
for you.

Moreover Jekyll supports customization in form of [plugins][JekyllPlugins]. So
you can extend it with already existing plugins or create a new one yourself.
Just keep in mind, that in the end Jekyll should generate, more or less, a set
of statically linked pages.

> **NOTE:** While Jekyll is extensible, if you want to use GitHub Pages to
host your content, [not all plugins might be supported](https://help.github.com/en/github/working-with-github-pages/about-github-pages-and-jekyll#plugins).

Jekyll has a bunch of assumptions built in. If you are creating a blog from
scratch then you should get familiar with [the directory structure used by
Jekyll.][JekyllStructure]


> **NOTE:** Jekyll is built with blogging use case in mind and [even supports
converting blogs from popular blogging platforms to Jekyll.](https://import.jekyllrb.com/)
So if you already have a blog you may try to convert it instead of starting
from scratch.

## Liquid

[Liquid][Liquid] is the template language that Jekyll uses. What is template
language? To understand that let's return to the problem of automatically
generating a list of posts in our blog.

A generator like Jekyll allows us to generate static content based on
description. For example, Jekyll may look at the list of files in the
[`_posts`][JekyllStructure] directory to find all the posts. However it still
lives the question of how the list of posts should be presented?

Potentially, Jekyll could have just hardcoded the way the list of posts should
be presented. However the creators of Jekyll went for a more customizable
approach.

You, the user of Jekyll, have to describe how the list of posts should look in
your favorite markup language by creating a template with a bunch of
placeholders. Jekyll will take the template and replace all the placeholders
with the actual list of posts and generate a complete page.

The language used to describe the template and placeholders inside the template
is called a template language. There are multiple different template languages
around, but the one used by Jekyll is Liquid.

Liquid is a rather reach template language that contains a bunch of different
operators, including conditions and loops.

Let's take a look at the following [markdown][Markdown] template for the list of
posts page using Liquid:

{% raw %}
```markdown
---
layout: default
---

{% for post in site.posts %}

# {{ post.title }}]

{{ post.excerpt }}

[Read More]({{ post.url }})

{% endfor %}
```
{% endraw %}

The document starts with so called [Front Matter][FrontMatter]. Front Matter
tells Jekyll how the file must be processed. I defer you to the Jekyll
documentation for the details on this.

What follows is a markdown document with a bunch of special directives inside
either {% raw %} `{{ }}` or `{% %}` {% endraw %}. Those are Liquid [objects and
tags](https://shopify.github.io/liquid/basics/introduction/) or, in other words,
placeholders that need to be populated by Jekyll.

Indeside Liquid objects and tags you can refer to a set of [variables that
Jekyll creates.][JekyllVariables] And specifically for the list of available
posts you need to refer to `site.posts`.

That's what the template above does - it iterates over the list of posts and
for each of the posts, Jekyll is instructed to generate a markdown snippet that
describes it. Jekyll documentation contains a bit more detailed description of
the variables available for posts [here.](https://jekyllrb.com/docs/posts/)

> **NOTE:** Liquid just defines the syntax of the template language. The
variables refered inside the template are provided by the Jekyll itself. So
hypothetically, if you decide to use a different generator instead of Jekyll
that happen to also use Liquid, you may find that while the overall template
syntax is similar the variables may change.

# Creating a new post

Creating a new post is as simple as adding a new file with a special name to the
`_posts` directory as described [here.](https://jekyllrb.com/docs/posts/) Jekyll
will be able to find the file and if it's correctly named and starts with the
correct Front Matter will identify it as a post.

# Conclusion

None of the information provided here is in any way new or hard to discover. It doesn't even go into great details about Jekyll, Liquid or GitHub Pages for that
matter.

The value of this post is twofold:

1. I need a post to test how it works, since I'm new to this.
2. It provides a structure that makes it easier for me personally to think
   about the whole thing. With this structure in my head I can use the
   documentation effectively.
