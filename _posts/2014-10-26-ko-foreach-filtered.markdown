---
layout: post
title: "Knockout ko filtered foreach"
date: 2014-10-26 21:30:00
comments: True
categories: JavaScript Knockout.js
---

In Knockout.js, you can bind multiple objects in an observable array to the same
HTML template using the ko foreach binding. It is very handy, but the code can become repetitive.
For example, when there are objects that we don't want to display in the array, we have to
have an extra computed array every time to take care of the filtered objects. It gets annoying
when you have to do it for every single view model. To save yourself some work, a custom
binding can be created to filter out the objects that you don't want to show in the view.
For our use case, we don't want to see the objects whose _destroy observable value is true.

{% highlight javascript %}

  ko.bindingHandlers.filterDestroyed =
    init: (element, valueAccessor, allBindings, viewModel, bindingContext) ->
      filteredValueAccessor = ->
        observableArray = valueAccessor()
        _.filter observableArray(), (viewModel) ->
          viewModel._destroy() is false

      ko.bindingHandlers["template"]["init"] element, ko.bindingHandlers["foreach"].makeTemplateValueAccessor(filteredValueAccessor)

    update: (element, valueAccessor, allBindings, viewModel, bindingContext) ->
      filteredValueAccessor = ->
        observableArray = valueAccessor()
        _.filter observableArray(), (viewModel) ->
          viewModel._destroy() is false

      ko.bindingHandlers["template"]["update"] element, ko.bindingHandlers["foreach"].makeTemplateValueAccessor(filteredValueAccessor), allBindings, viewModel, bindingContext

{% endhighlight %}

Also, don't forget to add the line

{% highlight javascript %}
  ko.virtualElements.allowedBindings.filterDestroyed = true
{% endhighlight %}

so that you can use the binding with virtual DOM elements!

P.S. I know that not everyone uses underscore.js, you can use Knockout's ko.utils.arrayFilter
to filter the array.
