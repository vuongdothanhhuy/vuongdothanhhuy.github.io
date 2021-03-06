---
layout: post
title: SailsJS Middleware to Validate Shopify Webhook
excerpt: "A Http Middleware to Validate Shopify Webhook in SailsJS"
modified: 2015-07-11
tags: [javascript, expressjs, web-development, nodejs, sailsjs]
comments: true
image:
  feature: banner.jpg
  credit: WeGraphics
  creditlink: http://wegraphics.net/downloads/free-ultimate-blurred-background-pack/
---

Hi there,

A few months ago, I developed a system using [SailsJS][sails] that works with Shopify Webhooks and saves data to a database. For those who do not know what is SailsJS: it is a NodeJS framework that wraps around ExpressJS and bundle a lot of things with ready-to-use configurations out of the box.

During the development process, there was a need to validate whether a webhook call is actually made by Shopify and from Shopify servers, and that its data has not been compromised. Luckily, it is found out that Shopify support [this][shopify-webhook]: one needs to simply compute the HMAC digest based on a share secret (get it in your Shopify dashboard) and the raw request body, then compares the result HMAC with the value in the `X-Shopify-Hmac-SHA256` header.

There are examples in Ruby and PHP on [Shopify documentation] [shopify-webhook], but none for NodeJS, specifically for SailsJS where things are little bit different (and magic!).

In SailsJS, there are `policy` and `middleware`. Normally, `policy` is used for authorization and access control, and so Shopify webhook validation would likely fall into this category. However, because `policy` is loaded after the `bodyParser` middleware, we cannot get the raw data for correct HMAC computation. Yes, I know it is still possible in a hacky way: modify the `bodyParser` middleware and [add back][raw-body] the `rawBody` varible to enable access to the raw body data.

I did not want to do the hack way, so I dig into middleware use. After some sort of researchs and try-and-error, it turned out that SailsJS uses ExpressJS middleware and that there is a way to obtain the raw body data without breaking the `bodyParser` middleware, so I came up with the following middleware that did the job:

{% highlight javascript %}
    ShopifyWebHookValidation: function (req, res, next) {
        var crypto = require('crypto');
        var SHOPIFY_APP_SHARED_SECRET = 'put_your_share_secret_here',
            SHOPIFY_HMAC_HEADER = 'X-Shopify-Hmac-SHA256',
            SHOPIFY_PATH = '/webhook/shopify',
            SHOPIFY_METHOD = 'POST';
        var shopifyHmacSha256 = req.get(SHOPIFY_HMAC_HEADER),
            path = req.path,
            method = req.method;

        if (shopifyHmacSha256 && path === SHOPIFY_PATH && method === SHOPIFY_METHOD) {

            //on Shopify route, POST and has hmac header
            req.hasher = crypto.createHmac('sha256', SHOPIFY_APP_SHARED_SECRET);

            req.on('data', function (chunk) {
                req.hasher.write(chunk);
            });

            req.on('end', function() {
                req.hasher.end();

                var hash = req.hasher.read();
                hash = new Buffer(hash).toString('base64');

                console.log('Hashes are:   ', hash, shopifyHmacSha256)

                if (hash === shopifyHmacSha256) {
                    req.isShopifyVerified = true;
                } else {
                    req.isShopifyVerified = false;
                }
            });
            next();
        } else {

            //not on Shopify route, skipping this middleware.
            next();
        }
    }
{% endhighlight %}

Here you can see that, because the request data comes as a stream, and usually the data stream is consumed by the `bodyParser` middleware, any middleware that comes after the selfish parser will not be able to read the stream anymore. Similarly, if we write a middleware to consume the stream and put it before the selfish parser, it will break that parser middleware. The reason is simple: we just consumed the data stream in out middleware, and there is nothing left for `bodyParser` to consume, so errors will occur. And yet, we cannot conditionally skip the `bodyParser` except hacking into its source, so that on Shopify webhook route, only our middleware runs and `bodyParser` is skipped.

The solution is, we 'subscribe' to the data stream so as the data come in, we grab it and save it to our local variable for use. The `data` event is emitted every time there is a new piece of data incomming, while the `end` event is emitted when the stream comes to an end and there is no more data to announce.

Calling `next()` should be done outside the event `end`, unlike some suggestions on Stackoverflow. The reason is simple: we just want to subscribe, then move on, and let the data flow from the stream to both our middleware and the `bodyParser`, as events are fired.

To use the middleware, in the `http.js` file, put the middleware in, while paying careful attention to the order of the middlewares, or else the Shopify middleware will refuse to work and eventually hang the whole ExpressJS request processing:

{% highlight javascript %}
    module.exports.http = {

      middleware: {

        order: [
          'startRequestTimer',
          'cookieParser',
          'session',
          'ShopifyWebHookValidation', //Put the middleware here, before bodyParser and after any session/cookie-related middleware.
          'bodyParser',
          'handleBodyParserError',
          'compress',
          'methodOverride',
          'poweredBy',
          '$custom',
          'router',
          'www',
          'favicon',
          '404',
          '500'
        ],

        // Place the middleware function here.
        // Or, if you prefer, you can place it in an external file and require it.

    };
{% endhighlight %}

As you can see, the Shopify middleware must be put `before` the `bodyParser` so that it can see the full and raw body content in a request. Putting it after `bodyParser` and it will fail to see the raw data, but the processed data returned from that middleware. It would still works and be able to compute the HMAC, but the result digest will be completely off comparing to the original HMAC header.

While we can put it even before the sessions and cookies middlewares, we would benefit from putting it after, so we can consume processed session/cookies data if necessary.

Hope this little post will be helpful for those struggling with the same problem.

[sails]: http://sailsjs.org/
[shopify-webhook]: https://docs.shopify.com/api/webhooks/using-webhooks#verify-webhook
[raw-body]: https://github.com/strongloop/express/issues/897