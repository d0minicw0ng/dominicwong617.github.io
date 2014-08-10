---
layout: post
title: "Open Jasmine specs from Vim"
date: 2014-08-10 14:55:00
comments: True
categories: Vim Jasmine
---

It sucks when you have to switch among your text editor, the terminal, and the browser.
It also sucks when you have to first run all the jasmine specs in a spec file before
drilling down to the specific spec that you really want to run.

The structure of a jasmine spec file usually goes like this:

{% highlight coffeescript %}

describe "Application.Views.Posts.IndexView", ->

  describe "render", ->
    jlet "view", -> new Application.Views.Posts.IndexView()

    beforeEach ->
      view.render()

    it "renders the 'Post' button", ->
      expect(view.$("button#post").text()).toEqual "Post"

    it "renders all the posts", ->
      expect(view.$(".post").length).toEqual 10

{% endhighlight %}

So let's say you want to run the spec "renders all the posts", how are you going to do that?
You will have to type
"http://localhost:3000/jasmine?spec=Application.Views.Posts.IndexView%20render%20renders%20all%20the%20posts"

Yes, you can first run the whole file and then click on the ones that you are interested in, but what if
there are 100 asynchronous specs in the file and your spec is the last one?

How annoying is that! It just sucks, and you know what I mean. Therefore, my co-worker and I wrote a vim script
to open jasmine specs from vim. When you type <Leader>oj, it looks at your current line and determine what specs
to run.

Let's say if your cursor is inside a "it" block or on the "it" line, then it will only run one spec. If your cursor
is inside a "describe" block, then it will run all the specs inside the "describe" block. If your cursor is at the
first line, then it will run all the specs in the file.

The vim script works by creating a string as the spec parameter in the URL. It does a reverse lookup on the lines
based on whether the line contains "describe"/"it" and keeps count of the lines' indentation levels. Finally, it opens
the URL from the terminal with the spec string.

You can find the script at [Github][github_url].

[github_url]: https://github.com/wizztjh/vim-open-jasmine-rice



