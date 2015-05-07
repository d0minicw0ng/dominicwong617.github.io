---
layout: post
title: "Serving Dynamic Contents Using CloudFront"
date: 2015-5-7 00:00:00
comments: True
categories: AWS CloudFront S3 Cache
---

OpsManager is an offline application and we have to store the data locally when the user has no access to the Internet.

The workflow goes like this:

1. The user has access to the Internet
2. He syncs the whole database from the server to IndexedDB
3. He goes somewhere with no Internet
4. Any changes made will be store locally
5. He comes back to a place with Internet
6. He syncs again. This time, it will only sync the changes that have been made, both locally and from the server.
7. If he resets the app, all data will be cleared, and the whole database will be synced from the server again.

We have been using Redis to store our caches and using `Rails.cache` to access the caches. It took on average 2.3 minutes
to sync down the database with a 8-Core iMac and 5.8 minutes with a 2-Core Samsung Chromebook (And what would happen
if there is a slow Internet connection? Even worse...). Pretty slow...

So, we have to speed this up so people don't waste hours staring at the computer. To fix this, our strategy is

1. Gzip the caches and push them to a S3 bucket
2. Serve the caches via CloudFront
3. IndexedDB object store put requests in batch

By moving the caches to AWS, the app will hit the server much less often and data will be served from a closer geographical location.

### Gzip the caches and push them to a S3 bucket

We take advantage of the Paperclip gem so that we don't have to handle the hassle
of uploading the caches to S3 ourselves. More importantly, it will automatically
store the caches in the local filesystem in the development environment.

First, we create a rails model called `ApiDatum` that has the attributes `name` (model name),
`json_file` (the data to be stored in S3) and `version` (a timestamp that stores the model's last updated at).
We use the gem `paperclip-deflater` for gzip.

{% highlight ruby %}
class ApiDatum < ActiveRecord::Base
  has_attached_file :json_file,
    styles: {
      gzip: {
        processors: [:gzip],
        s3_headers: { content_type: 'text/plain', content_encoding: 'gzip' },
      }
    },
    s3_protocol: 'https',
    s3_permissions: :private
  do_not_validate_attachment_file_type :json_file
end
{% endhighlight %}

We recreate the caches every hour by running a job called `ApiCacheJob`. The `generate` method recreates
the cache and updates the ApiDatum's `json_file` and `version`.

{% highlight ruby %}
def generate model_name
  version = get_collection_version model_name
  tempfile = Tempfile.new model_name

  begin
    tempfile.write cached_collection_json(model_name)
    tempfile.rewind
    api_datum = ApiDatum.find_or_create_by name: model_name
    api_datum.json_file = tempfile
    api_datum.json_file.instance_write :file_name, "#{model_name}_#{Time.now.to_i}"
    api_datum.version = version
    api_datum.save
  ensure
    # NOTE: passing true as the argument will unlink the temp file
    tempfile.close true
  end
end
{% endhighlight %}

That's it for pushing the gzipped caches to S3. Of course, you will also need a S3 bucket set up already.
You have to update the bucket's CORS configuration to make sure that only your application can make requests
to the data. Your CORS configuration should look something like this.

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<CORSConfiguration xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
  <CORSRule>
    <AllowedOrigin>Your domain</AllowedOrigin>
    <AllowedMethod>GET</AllowedMethod>
    <MaxAgeSeconds>3000</MaxAgeSeconds>
    <AllowedHeader>*</AllowedHeader>
  </CORSRule>
</CORSConfiguration>
{% endhighlight %}

### Serve the caches via CloudFront

Next, you will have to create a CloudFront distribution to serve the caches.
Make sure that when you create a CloudFront distribution, choose `Yes` for
`Restrict Bucket Access` and choose `Yes, Update Bucket Policy` for
`Grant Read Permissions on Bucket` so that people can't access the content
without the keys and it will automatically update S3's policy to this.

{% highlight xml %}
{
  "Version": "2008-10-17",
  "Id": "PolicyForCloudFrontPrivateContent",
  "Statement": [
    {
      "Sid": "1",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity YOUR_IDENTITY"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR_BUCKET/*"
    }
  ]
}
{% endhighlight %}

You will also need to generate a CloudFront Key Pairs at `Your Security Credentials`
to create signed URLs to access the caches.

We have a helper method in `ApiDatum` that generates the signed URL for CloudFront with the `cloudfront-signer` gem.

`config/initializers/cloudfront-signer.rb`
{% highlight ruby %}
if Rails.env.production? || Rails.env.staging?
  AWS::CF::Signer.configure do |config|
    config.key = ENV["CLOUDFRONT_KEY"]
    config.key_pair_id  = ENV["CLOUDFRONT_KEY_PAIR_ID"]
    # NOTE: The signed URL expires in 300 seconds
    config.default_expires = 300
  end
end
{% endhighlight %}

`app/models/api_datum.rb`
{% highlight ruby %}
def sign_url
  if Rails.env.production? || Rails.env.staging?
    AWS::CF::Signer.sign_url "#{ENV["CLOUDFRONT_DOMAIN"]}/#{json_file.path(:gzip)}"
  else
    # NOTE: If it is development environment, just return the filesystem path
    json_file.url
  end
end
{% endhighlight %}

### IndexedDB object store put requests in batch

When the application is being reset, the web worker sends a HTTP request to the server to request for
the CloudFront URLs to fetch from.

`app/controllers/files_controller.rb`
{% highlight ruby %}
def full_sync_collection
  api_data = ApiDatum.all.inject({}) do |memo, datum|
    memo[datum.name] = [ { version: datum.version, url: datum.sign_url } ]
    memo
  end

  render json: api_data, status: :ok
end
{% endhighlight %}

It returns a JSON that has each model's version and URL to fetch. (The value is an array because
we also serve location based caches with multiple URLs, but we won't discuss it here)

The worker will then go fetch all the caches and put them into IndexedDB in batch. If the IDBTransaction
fails, it will retry the transaction once. The `syncObjectStore` method returns a `Promise` and is pushed
to a promises array. When all the promises are resolved, the sync is complete.

The operation has to be in batch because IndexedDB becomes slower and slower as the transaction's size grows.

{% highlight coffeescript %}
# NOTE: @tableName is the object store name, and db is the opened IndexedDB
syncObjectStore: (itemsToBeSaved, db) ->
  new Promise (resolve, reject) =>
    putNext = =>
      if itemsToBeSaved.length > 0
        batchItems = []
        transaction = db.transaction [@tableName], "readwrite"
        objectStore = transaction.objectStore @tableName

        times = Math.min itemsToBeSaved.length, 10000
        for i in [0...times]
          item = itemsToBeSaved.pop()
          batchItems.push item
          objectStore.put item

        transaction.oncomplete = putNext

        retry = =>
          retryTransaction = db.transaction [@tableName], "readwrite"
          objectStore = retryTransaction.objectStore @tableName
          for item in batchItems
            objectStore.put item
          retryTransaction.oncomplete = putNext
          retryTransaction.onerror = (event) -> reject event
          retryTransaction.onabort = (event) -> reject event

        transaction.onerror = -> retry()
        transaction.onabort = -> retry()

      else
        resolve()

    putNext()
{% endhighlight %}

### Result

| Computer                     | Before      | After        |
| ---------------------------- | ----------- | ------------ |
| iMac (8-Core)                | 2.3 minutes | 31.4 seconds |
| Samsung Chromebook (2-Core)  | 5.8 minutes | 4.2 minutes  |

It takes very little time (~10 seconds for 350+ HTTP requests) to fetch everything from CloudFront
and the majority of time spent goes to putting the objects into IndexedDB. There is a bottleneck
with IndexedDB and maybe this is the next thing to fix :).

We also face hardware limitations because many of our clients use very very old computers. (Our clients
are from places like Indonesia, Papua New Guinea or Iraq, where people don't normally own the newest computers)
We will have to push our clients to use slightly more modern computers :)

PS. You can use `window.navigator.hardwareConcurrency` to determine the system's total number of logical processors available to the user agent.
