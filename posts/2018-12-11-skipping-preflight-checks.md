<h1 class="post-heading">Skipping preflight checks</h1>
<em>December 11, 2018 - [@kddeisz](https://twitter.com/kddeisz)</em>

Preflight checks are part of the CORS system that browsers fire before cross-origin requests in order to determine if the request is allowed. These requests can add up over time and cause a discernable lag in your web application.

If the architecture of your application involves a frontend JavaScript SPA served from one domain and a backend API server served on another, then CORS requests are your reality. For example, our frontend is served from AWS CloudFront on `https://platform.culturehq.com`, while our API is served from `https://api.culturehq.com`.

However, if these resources are served on the same parent domain (in this case `culturehq.com`), there is a way to skip preflight checks entirely. Below is a discussion of how that can happen, as well as a bit of background on the topics mentioned so far.

## CORS

[CORS](https://developer.mozilla.org/en-US/docs/Glossary/CORS) is a system built with HTTP headers that is used to determine whether or not JavaScript code is allowed to access resources on certain servers. By default, due to the [same-origin security policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy), JavaScript code cannot request resources on servers that do not match the current origin (as determined by `document.domain`). However, by configuring CORS headers you can open up certain resources to certain HTTP methods and headers, thereby allowing you server to be accessed from various cross-origin JavaScript resources.

## document.domain

When browsers are determining whether or not to issue a preflight check, they'll check the `document.domain` property. If the value matches up with the domain of the requested resource, a preflight check doesn't get issued. You can [change this value](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy#Changing_origin) at any time from within JavaScript code to be either the current domain or any superdomain of the current domain (for instance, on the `platform.culturehq.com` domain you can set it to `culturehq.com`). Then, if you're requesting resources on the other domain preflight checks will be skipped. The problem still exists however if you're trying to access a separate subdomain (as in `api.culturehq.com`). In this case you'll need to change the domain on both the server and the client.

## Aligning the API

In order to change the domain of the server such that preflight checks will be skipped entirely across subdomains, we can get clever. The first step is to configure an endpoint on the server that will return a very simple HTML response that includes the JavaScript to set the domain to the superdomain. The example below configures a `Ruby on Rails` application to serve up just that kind of page:

```ruby
PROXY_RESPONSE = <<~HTML
  <!DOCTYPE html>
  <html>
    <body>
      <script>document.domain = 'culturehq.com'</script>
    </body>
  </html>
HTML

Rails.application.routes.draw do
  get :proxy, to: lambda { |_env|
    [200, { 'Content-Type' => 'text/html' }, [PROXY_RESPONSE]]
  }
end
```

In the above example, we've configured the `/proxy` endpoint to return an HTML document that will immediately set the `document.domain` property to `culturehq.com`. Then, any requests made from this page will pass the same-origin check if they're also issued from the `culturehq.com` domain.

## Aligning the frontend

On the frontend, we can then embed an `iframe` within the current page that requests that proxy page, and steal the `fetch` function from it in order to issue requests to our API.

```javascript
const fetcher = { fetch: window.fetch.bind(window) };

const skipPreflightChecks = () => {
  const iframe = document.createElement("iframe");
  iframe.onload = function () {
    const { fetch } = this.contentWindow;
    fetcher.fetch = fetch.bind(this.contentWindow);
  };

  iframe.setAttribute("src", "https://api.culturehq.com/proxy");
  iframe.style.display = "none";

  document.domain = "culturehq.com";
  document.body.appendChild(iframe);
};
```

Using the above code we can then call `skipPreflightChecks()`, and once it resolves we can use `fetcher.fetch` to issue requests that pass the same origin check.

## tl;dr

Preflight checks are triggered because of the same-origin check. You can change the domain of your webpage by modifying `document.domain`. Given these facts, you can embed an iframe in your webpage that sets its own domain to your top-level domain as well as setting your domain to your top-level domain in your frontend to bypass these checks and achieve preflight-less cross subdomain requests that are faster to resolve.

[Back to home](/)
