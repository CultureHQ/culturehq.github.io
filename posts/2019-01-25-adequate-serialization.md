<h1 class="post-heading">adequate_serialization</h1>
<em>January 25, 2019 - [@kddeisz](https://twitter.com/kddeisz)</em>

At CultureHQ, we've developed [adequate_serialization](https://github.com/CultureHQ/adequate_serialization), a Ruby object JSON serialization library compatible with Rails. We've been running it in production for a little over a year now, and aside from a few special cases it performs all of the serialization in our API responses.

It's differentiated from other serialization solutions in that when combined with Rails it is cache-first, taking advantage of the `fetch_multi` functionality baked in. Below is the story of how it came to be, and how you can use it in your applications.

## Previous attempts

At CultureHQ, we made the decision early to split the front and back ends of our web application. We assumed that as our technology and product offering continued to grow, we would need a dedicated API server without presentational concerns that we could access from various client interfaces.

This separation necessitated that we have a commonly understood language between the client and server, as well as various idiomatic consistencies so that the API responses were easily understood. After looking at all of the [options for Rails serialization](https://www.ruby-toolbox.com/categories/API_Builders) (of which there are many) we settled on [active_model_serializers](https://github.com/rails-api/active_model_serializers), and began working on implementing it in our application.

For a while `ams` worked well for us, but we ended up running into issues when working with caching. If you had minor differences you ended up having to cache entirely separate objects. Additionally, the `ams` docs themselves state that [caching doesn't improve performance](https://github.com/rails-api/active_model_serializers/issues/1586). Finally, it ended up hitting the cache many times for large relations, which ended up slowing things down more than helping depending on network latency.

[← Back to home](/)
