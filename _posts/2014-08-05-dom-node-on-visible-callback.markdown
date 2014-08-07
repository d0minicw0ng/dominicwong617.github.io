---
layout: post
title: "DOM Node on visible callback"
date: 2014-08-05 00:00:00
categories: JavaScript jQuery
---

Context:

At [OpsManager][opsmanager_url], we have been having memory issues with our Daily Drilling Reports
module because we used to have to store hundreds of megabytes of data into Backbone Collections.

Since OpsManager is an offline application, we had to somehow get a hold of the data without having
to talk to the server. We use IndexedDB to store the data before syncing to the server (when there
is an internet connection) and we use Backbone.js as the frontend MVC framework.

However, Backbone.js is not the best solution for our problem because of the extra amount of data
that we are using. While we could just fetch data from IndexedDB when we need it, we are just
letting the Backbone Collections to linger around when it is not needed.

The goal of this refactor is to abandon Backbone Collections altogether and use a data store service
to talk to IndexedDB and only fetch the data needed when we navigate to specific views. If I go to
the page of a specific Daily Drilling Report, I only want one object, instead of the complete collection,
which is currently the size of ~75,000. We forgo Backbone Model/Collections and replace
the model with Knockout.js in Backbone View.


Problem Encountered:

To show the drilling activity form which is invisible on page load, the user needs to click the
add drilling activity icon in the Daily Drilling Report tab. A drilling activity form will pop up
with a spinner because the start time and the start depth of a Daily Drilling Report need to be loaded
from the data store. The start time input field has a timepicker (jQuery.timepicker), which has to be
initialized when we enter the Backbone View. But the input field is not visible until the start time and
start depth are loaded, what can I do? The simplest way is to wait until the start time input field is visible
and then call initialize timepicker. I came up with a custom jQuery binding that would solve this problem.

{% highlight javascript %}

/**
* function $.fn.onVisible runs callback function once the specified element is visible.
* callback: A function to execute at the time when the element is visible.
* example: $(selector).onVisible(callback);
*/

(function($) {

    $.fn.onVisible = function (callback) {
        var self = this;
        var selector = this.selector;

        if (self.is(":visible")) {
            callback.call(self);
        } else {
            timer = setInterval(function() {
                if ($(selector).is(":visible")) {
                    callback.call($(selector));
                    clearInterval(timer);
                }
            }, 50);
        }
    }
}(jQuery));

{% endhighlight %}

The selected DOM node will call the callback function if it is already visible. If not, a time interval
will be created to check if it is visible until so. Then, the callback function will be called and the
time interval will be cleared.

[opsmanager_url]: http://opsmanager.com
