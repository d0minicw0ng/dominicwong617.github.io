---
layout: post
title: "jQuery.uploadPreviewer.js"
date: 2015-4-17 00:00:00
comments: True
categories: JavaScript jQuery File
---

The default HTML file input is pretty restrictive. You try to click on the file input and select some files.
The next time you click on it and try to select more files, it will replace all the previously selected files.
You can't just add or remove files. OUCH. You can't really set the files programmatically. It's an understandable
behavior because there is a security issue that we have to consider. What if you can set the file input's value programmatically?
Then your `/etc/passwd` file will be uploaded to a remote server without you even knowing :) However, you can work
around this restriction by having a separate array that stores the files to be posted. Every time you select some files,
the files will be added to this array.  If you want to remove any files, you simply remove them from the array.
When you actually make your POST request, you will send this array instead of the default FileList to the server.

My colleague [Rui][rui_url] and I have written a jQuery plugin to allow adding and removing selected files
without having to write all the codes yourself. All you have to do is

{% highlight javascript %}

var fileUploader = $*('input[type=file]').uploadPreviewer();
// Do some work....
fileUploader.submit(successCallback, errorCallback);

{% endhighlight %}

The source code is located [here][plugin_url] and the demo is [here][demo_url]


[rui_url]: https://github.com/rpbaltazar
[plugin_url]: https://github.com/opsmanager/jquery.uploadPreviewer.js
[demo_url]: http://opsmanager.github.io/jquery.uploadPreviewer.js
