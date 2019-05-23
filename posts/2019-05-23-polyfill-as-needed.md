<h1 class="post-heading">Polyfill as needed</h1>
<em>May 23, 2019 - [@kddeisz](https://twitter.com/kddeisz)</em>

Because browsers implement their own versions of JavaScript, they don't all support the same functions. For example, `Array#includes` is supported in all of the major browsers except Internet Explorer. So, if you include that function in your application (or one of your dependencies does) your application will break in Internet Explorer only.

A common solution to this problem is a polyfill. For example, a polyfill for the `Array#includes` function would look like:

```javascript
if (!Array.prototype.includes) {
  Array.prototype.includes = function includes(searchElement) {
    return this.indexOf(searchElement) !== -1
  }
}
```

Frequently, applications will ship a whole set of polyfills (for example [core-js](https://www.npmjs.com/package/core-js)) alongside the main application code in order to allow the application to run on all of the browsers that it supports. This works, but ends up shipping a lot of unnecessary code to the majority of users just to support the small minority that are running the older browsers.

## Solutions

Fortunately, some smart people have come up with solutions to this problem. [polyfill.io](https://polyfill.io/v3/) is a service from Financial Times that will conditionally return polyfills based on the user agent of the requesting browser. So, you can visit [https://polyfill.io/v3/polyfill.js](https://polyfill.io/v3/polyfill.js) in your browser to see what would be returned.

The only problem with this service is that it has no SLA, so most businesses can't rely on it. In light of this, Kent C. Dodds [wrote a blog post](https://kentcdodds.com/blog/polyfill-as-needed-with-polyfill-service) about using `polyfill.io`'s [polyfill-library](https://github.com/Financial-Times/polyfill-library) to build your own polyfill service.

In the spirit of Kent's post, we've created our own service for polyfilling, and the code can be found [here](https://github.com/CultureHQ/polyfill-lambda).

## `polyfill-lambda`

`polyfill-lambda` is a service that can be deployed to Amazon Web Services to handle all your polyfilling needs. It works by building three main resources in AWS:

- a [CloudFront](https://aws.amazon.com/cloudfront/) distribution - this is a CDN (Content Delivery Network) that handles caching and distributing your polyfills around the world at very quick speeds
- an [S3](https://aws.amazon.com/s3/) bucket - this is used as the origin for your CloudFront requests
- a [lambda](https://aws.amazon.com/lambda/) function - this is where the work happens. This lambda function will intercept requests going to S3 and instead will return an appropriate polyfill based on the requesting user agent

You can think of the setup as basically a static website hosted on S3 that is delivered through CloudFront. The added twist is that when a page that isn't cached is required a lambda function is triggered and run instead of CloudFront fetching the file from S3.

With these resources combined, you can take it a step further by requesting a certificate from AWS's [certificate manager](https://aws.amazon.com/certificate-manager/) and point your a route from a hosted zone in [Route53](https://aws.amazon.com/route53/) to it with an alias record. What that means is that provided you have some `foo.com` domain hosted on AWS, you can have a `polyfill.foo.com` polyfilling service for next to nothing in terms of cost (you'd have to rack up massive numbers of requests for this to cost anything).

## Security

As with all code that gets executed in your browser, you should be very careful with sources that you trust. If you're going to be adding a whole other domain from which your application can pull scripts, you should make sure to think about the attack surface area that you're adding.

Fortunately modern browsers have a way of protecting our applications in the [Content-Security-Policy](https://scotthelme.co.uk/content-security-policy-an-introduction/) header. There are a lot of great resources on the web to learn about CSP, so this post won't get into the details. Suffice to say when you add the polyfill service to your application be sure to also add that domain to the `script-src` of your CSP.

## tl;dr

We build an AWS lambda function called [polyfill-lambda](https://github.com/CultureHQ/polyfill-lambda) that provides a service that polyfills only the necessary functions for the requesting browsers.

[← Back to home](/)
