---
layout: post
title: "Make Your Test Suite Run Faster"
date: 2015-10-13 14:05:00
comments: True
categories: Ruby RSpec Tests
---

OpsManager's test suite is too slow. It takes ~6 hours to run the integration tests
and ~1 hour to run the RSpec unit tests. Below are some tips to make your tests run faster.

We managed to bring integration tests execution time down from 6 hours 2 minutes to
1 hour 45 minutes and RSpec unit tests execution time from 1 hour 10 minutes to 11 minutes.
Here are a few things for you to consider in order to make your tests run faster.

### Tip 1: Capybara.default_wait_time

If you have `Capybara.default_wait_time` set to something
much higher than the default (2 seconds), revert it. It will waste a lot of time because
Capybars will wait for the specified wait time every time when an element doesn't appear.
If you encounter situations where you must wait for more than 2 seconds, use
{% highlight ruby %}
Capybara.using_wait_time(n) do
  # your test
end
{% endhighlight %}

### Tip 2: Capybara has vs has_no selector

If you want to assert that a DOM node is not in the page, use `has_no_*`
instead of `has_*` selector because `has_no_*` will not wait for an element to 'not' appear and will pass/fail as soon as it runs.

### Tip 3: Disabling config.threadsafe

Making `let` not threadsafe will make it a bit faster. [Reference][relish_url]

[relish_url]: https://www.relishapp.com/rspec/rspec-core/docs/helper-methods/let-and-let

### Tip 4: Defer Garbage Collection

Run garbage collection less frequently. [Reference][gc_url]

[gc_url]: https://ariejan.net/2011/09/24/rspec-speed-up-by-tweaking-ruby-garbage-collection/

### Tip 5: Use different database cleaning strategies for integration/unit tests

Use `truncation` for integration tests that involves JavaScript and `transaction` for unit tests.
Use `transactional_fixtures` for unit tests. Do not use `transactional_fixtures` for integration tests.
In our case, it depends on whether the test involves using IndexedDB.

{% highlight ruby %}
config.include SeleniumHelper

config.before(:each) do |example|
  strategy = example.metadata[:indexeddb] ? :truncation : :transaction
  DatabaseCleaner.strategy = strategy
  DatabaseCleaner.start
end
{% endhighlight %}

{% highlight ruby %}
module SeleniumHelper
  def self.included(base)
    base.class_eval do
      self.use_transactional_fixtures = metadata[:indexeddb] ? false : true
    end
  end
end
{% endhighlight %}

### Tip 6: Avoid using FactoryGirl.create

`FactoryGirl.create` persists the instance of the model while `FactoryGirl.build` keeps it only in memory. [Reference][stack_overflow_url]

[stack_overflow_url]: http://stackoverflow.com/questions/14098031/whats-the-difference-between-the-build-and-create-methods-in-factorygirl

### Tip 7: parallel_tests gem

Use parallel_tests to run your specs in parallel. However, you will run into problems.
To solve `Too many open files`, you can increase the max number of open file descriptors
by running `ulimit -n your_number` or changing `kern.maxfiles` and `kern.maxfilesperproc`.
To solve `no such session`, you should not set `Capybara.server_port` and allow Capybara
to choose whichever ports available.


