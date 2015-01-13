---
layout: post
title: "I suck and my tests are order dependent"
date: 2015-01-13 22:50:00
comments: True
categories: Ruby
---

While we are working hard to upgrade OpsManager's Ruby on Rails version to 4.2, we
found the need to either dive deep into the (not maintained since May 2013) state_machine gem
or switch to using aasm (previously acts_as_state_machine).

During the process of running the state_machine specs, we found something really funny.
In MiniTest::Unit::TestCase, there is a method called
{% highlight ruby %} i_suck_and_my_tests_are_order_dependent! {% endhighlight %}

What it does is

{% highlight ruby %}
Call this at the top of your tests when you absolutely positively need to have ordered tests.  In doing so, you are admitting that you suck and your tests are weak.
{% endhighlight %}

There also happens to be an issue about this method name on [GitHub][github_url].
That made my day :)

[github_url]: https://github.com/seattlerb/minitest/issues/398
