---
layout: blogpost
title: Refresh a single Turbo frame
subhead: When you want to reload from the backend, but not the whole page
excerpt: "Turbo stream's refresh action is great as it allows refreshing user dependent views. The downside? The whole page is refreshed. Sometimes all you need is one frame to be refreshed."
categories: code
date: 2024-12-12 10:45:00 +0200
imgclass: fresh
---

In one of my projects, I'm using the latest version of [Hotwire](https://hotwired.dev/) in combination with the [turbo-rails](https://github.com/hotwired/turbo-rails/) gem.

One of the actions Turbo Stream implements natively is `refresh`, and in particular I want to talk about the "morph" version of refresh.

If you respond with (or broadcast) a `refresh` action from the backend, the frontend will trigger a full page reload by doing a HTTP request to the backend. And then, depending if you use the "morph" behavior or not, it will either replace the full document body, or it will "morph" it by trying to cleverly replace only the parts that changed.

You can read more about it in the Hotwire [documentation](https://turbo.hotwired.dev/handbook/page_refreshes) or on the various [blog](https://radanskoric.com/articles/turbo-morphing-deep-dive) [posts](https://jonsully.net/blog/turbo-8-page-refreshes-morphing-explained-at-length/) describing how morphing works.

However, in practice I find the "do a full page refresh and morph the content" behavior problematic and often too heavy handed for various reasons:

One is, stimulus controllers who hold state need to be made aware that they can be morphed and manage their state appropriately (see [this thread](https://github.com/hotwired/turbo/issues/1083) as an example). This can be quite a lot of fragile code based on `turbo:morph` events.

Another thing is, you need to be _really_ careful to mark every part of that page that you *don't* want to morph with `data-turbo-permanent` attributes. This can be tricky if your views are complex and hold many partial rendering. It can also be impractical if you have some wrapper behavior that shouldn't be morphed and then later on, you want some inside part to actually be morphed.

And finally, it's a full page load, which can be unnecessarily heavy on your backend if you know the update will be restricted to a limited part of the page.

It has to be said though, one thing that is great about the `refresh` action is, by telling the browser to do a backend request to fetch fresh data, it lets you update parts of the view that are user-dependent. Which wouldn't be realistic to do with the `update` or `replace` Turbo stream actions (you would have to broadcast one event per user).

All in all, what I want is: encapsulate part of the template with a Turbo frame, and then broadcast an event that tells the browser to refresh only that frame (via a backend request). How can we do this?

### Tutorial time

Let's take an example to illustrate why I want single-frame refresh instead of the whole page refresh. Imagine I have a simple e-shop with an index of `products`. Each `product` can have a limited `stock` meaning that the customer can only add up to that number of `items` to their `basket`. For example, if the stock of avocados is 5, the customer can add 5 avocados to their basket. Once they reach 5, the UI changes and they can't add more.

Here are the main models:

```ruby
class User < ApplicationRecord; end;

class Product < ApplicationRecord
  has_many :items, dependent: :destroy
end

class Basket < ApplicationRecord
  belongs_to :user
  has_many :items, dependent: :destroy
end

class Item < ApplicationRecord
  belongs_to :basket
  belongs_to :product

  def stock_limit_reached?
    product.stock <= quantity
  end
end
```

And before we define controller endpoints, here is the products index when a user has added 4 avocados to their basket:

![Wireframe of a simple e-shop with 8 fruits for sale](/img/posts/eshop.jpg)

And if they add a 5th Avocado they can't add any more:

![Wireframe of a simple e-shop with 8 fruits for sale, avocado is disabled](/img/posts/eshop-stock-limit.jpg)

We can also click on one product and see more details about the product and also add to the basket from that page:

![Wireframe of a simple e-shop page of an Avocado](/img/posts/avocado.jpg)

Let's focus on what the view templates for the Product index & show pages would look like. As we can see, we have views that depend on the current user's basket:

```ruby
class ProductsController < ApplicationController
  def index
    @products = Product.all
    @basket_items = current_user.basket.items.group_by(&:product_id)
  end

  def show
    @product = Product.find(params[:id])
    @item = current_user.basket.items.find_by(product: @product)
  end
end
```

```erb
<%# app/views/model/products/index.html.erb %>
<% @products.each do |product| %>
  <% item = @basket_items[product.id] %>
  <%= render "products/product", product:, item: %>
<% end %>
```

```erb
<%# app/views/model/products/_product.html.erb %>
<div class="card">
  <h1><%= product.name %></h1>
  <div>stock: <%= product.stock %></div>
</div>
<% if item&.stock_limit_reached? %>
  <div class="button-muted">no more</div>
<% else %>
  <%= button_to "Add one", items_path, params: {product_id: product.id}, method: :post %>
<% end %>
```

```erb
<%# app/views/model/products/show.html.erb %>
<h1>@product.name</h1>
<div>
  <p><%= @product.description %></p>
  <div>Origin: <%= @product.origin %></div>
  <div>stock: <%= @product.stock %></div>

  <% if @item&.stock_limit_reached? %>
    <div class="button-muted">no more</div>
  <% else %>
    <%= button_to "Add one", items_path, params: {product_id: @product.id}, method: :post %>
  <% end %>
</div>
```

As we can see in those templates, every time the user clicks the `Add one` button, it increments the quantity of this product in their own basket. Here is a simple controller endpoint to do that:

```ruby
class ItemsController < ApplicationController
  def create
    @item =
      current_user
        .basket
        .items
        .find_or_initialize_by(product_id: params[:product_id])
    @item.quantity += 1
    @item.save

    # now what?
  end
end
```

The response of that request is the whole point of this blog post. What are our options?

Without using Turbo, we can simply `redirect_back`. Now let's imagine that these pages are already using Turbo and there is more complexity to them and we can't simply `redirect_back`.

We could try to render a Turbo stream `replace` action but we have a problem with this: the view depends on the current user (via the `item` record). And this `item` is not necessarily present in the basket prior to the request (in which case it would be `nil`) so we can't have a Turbo frame whose target depends on the `item`.

We could render a Turbo stream `refresh` action (with or without morphing). It will reload the whole page. For the Product show page, that could be acceptable, but it's still pretty wasteful. Let's imagine a real e-shop: the "My basket" area may be complex to compute, as could other parts of the page be (recommendations of other similar products, etc). And for the index of products, it's even more wasteful: there could be tens, or hundreds of products on that page, as well as other expensive to compute sections.

There are really only 2 things that needs refresh here: the "My basket" composition - which we are ignoring in this tutorial - and the button area for the product that was just added: if stock limit is reached, we need to disable it.

For the sake of simplicity, I will encapsulate the whole product in a Turbo frame on both index and show views:

```erb
<%# app/views/model/products/_product.html.erb %>
<%= turbo_frame_tag product %>
  <div class="card">
    <h1><%= product.name %></h1>
    <div>stock: <%= product.stock %></div>
  </div>
  <% if item&.stock_limit_reached? %>
    <div class="button-muted">no more</div>
  <% else %>
    <%= button_to "Add one", items_path, params: {product_id: product.id}, method: :post %>
  <% end %>
<% end %>
```

```erb
<%# app/views/model/products/show.html.erb %>
<%= turbo_frame_tag @product %>
  <h1>@product.name</h1>
  <div>
    <p><%= @product.description %></p>
    <div>Origin: <%= @product.origin %></div>
    <div>stock: <%= @product.stock %></div>

    <% if @item&.stock_limit_reached? %>
      <div class="button-muted">no more</div>
    <% else %>
      <%= button_to "Add one", items_path, params: {product_id: @product.id}, method: :post %>
    <% end %>
  </div>
<% end %>
```

What I want from the ItemsController create endpoint is to render a Turbo stream action that reloads the one frame associated with the product that was added to basket.

One feature that was added [recently](https://github.com/hotwired/turbo/pull/1192) to Turbo is the option to `reload` a Turbo frame. This requires the frame to have a `src` attribute. When the `reload()` function is triggered, it will re-fetch from the `src` url and replace the frame with the response. And the frame itself can also have a `refresh="morph"` attribute which will do the replacement with the "morph" behavior.

One thing to keep in mind: the `src` url _must_ respond with a template containing a Turbo frame with the same target as the caller. so for example if we have this frame in a template:

```erb
<%= turbo_frame_tag "basket", src: basket_path %>
```

Then the `basket_path` endpoint must respond with a template like this:

```erb
<%# src is optional here, original will be kept %>
<%= turbo_frame_tag "basket" %>
  <h3>🛒 My basket</h3>
  <%# [...] %>
<% end %>
```

That's exactly what I want to do. Let's implement it!

First of all we need a custom Turbo stream action in our JavaScript:

```js
Turbo.StreamActions.reload = function () {
  // if the frame has a `src`, reload
  // if not but has a data-src => load that src
  // else do nothing
  document.querySelectorAll(`turbo-frame#${this.target}`).forEach((frame) => {
    if (frame.src) {
      frame.reload();
    } else if (frame.dataset.src) {
      frame.src = frame.dataset.src;
    }
  });
};
```

As you can see it does a bit more than just reloading. The problem with adding a `src` to the Turbo frame, is that Turbo will consider this frame eager loaded and will do a backend request to the `src` url. As far as I know, there is no way to tell the frame that it is already loaded. So if we don't want to use eager loading and render the Turbo frame content on first load, setting a `src` would be wasteful and doing too many backend requests for nothing. But we can use `data-src` in that case, and on receiving the `reload` action, it will either execute `reload()` on the frame if there is a `src`, else if there is a `data-src`, it pushes the `data-src` value to `src` which triggers a load. If neither `data-src` nor `src` are set, it does nothing (can't reload if there is no url at all).

We can add a helper to call this custom action from the backend code. This is optional but makes the code easier to read:

```ruby
# app/helpers/turbo_stream_actions_helper.rb
module TurboStreamActionsHelper
  def reload(frame_id)
    turbo_stream_action_tag :reload, target: frame_id
  end
end
Turbo::Streams::TagBuilder.include(TurboStreamActionsHelper)
```

Let's use this helper to render the action from our controller:

```ruby
class ItemsController < ApplicationController
  def create
    @item =
      current_user
        .basket
        .items
        .find_or_initialize_by(product_id: params[:product_id])
    @item.quantity += 1
    @item.save

    respond_to do |format|
      format.turbo_stream do
        render turbo_stream: turbo_stream.reload(@item.product)
      end
    end
  end
end
```

Cool! We just told the browser "reload the frame whose target is `@item.product`". Now we need to add the `src` or `data-src` attributes to the Turbo frames.

As we can see, this necessitates two new GET endpoints, one that renders the product template for the index and one for the show.

Luckily (mind blown, I totally didn't anticipate this 😉), the `products/show.html.erb` template is exactly what the `src` request must respond with, and in the same way, the `products/_product.html.erb` partial is what needs to be rendered for the index product card reload.

All we need to do is to create a new endpoint for that last one:

```ruby
class ProductsController < ApplicationController
  # GET card_product_path defined in routes
  def card
    @product = Product.find(params[:id])
    @item = current_user.basket.items.find_by(product: @product)
  end
end
```
```erb
<%# app/views/products/card.html.erb %>
<%= render "products/product", product: @product, item: @item %>
```

And we can now set the `src` or `data-src` to our frames. For the sake of completion, let's turn the index ones into eager loaded frames and keep the show one into a preloaded frame:

```erb
<%# app/views/model/products/index.html.erb %>
<% @products.each do |product| %>
  <% item = @basket_items[product.id] %>
  <%= turbo_frame_tag product, refresh: "morph", src: card_product_path(product) %>
<% end %>
```

```erb
<%# app/views/model/products/show.html.erb %>
<%= turbo_frame_tag @product, refresh: "morph", data: {src: product_path(@product)} %>
  <h1>@product.name</h1>
  <div>
    <p><%= @product.description %></p>
    <div>Origin: <%= @product.origin %></div>
    <div>stock: <%= @product.stock %></div>

    <% if @item&.stock_limit_reached? %>
      <div class="button-muted">no more</div>
    <% else %>
      <%= button_to "Add one", items_path, params: {product_id: @product.id}, method: :post %>
    <% end %>
  </div>
<% end %>
```

And tada 🎉 it all works as expected!

### Going further

One thing to note is that the Turbo stream `refresh` action is _special_ in the sense that by default, it doesn't refresh the page the request comes from and you have to specify `request_id: nil` to actually refresh your own page. This is done to avoid double refreshes when the action comes from a broadcast via WebSockets.

With our own reload action, no such thing is implemented, so you have to be careful avoiding double reloads yourself.

And what if we wanted to implement the broadcast version of this? Let's imagine we have a new use case where the stock of one product is diminished. Let's say when a user buys a product, its stock automatically diminishes by the amount bought.

Now we need to propagate the reload of the related product to all users of the platform. We can do that with turbo-rails and a broadcast action:

```ruby
class Product < ApplicationRecord
  has_many :items, dependent: :destroy

  # Say this method is called after a purchase is completed
  def decrease_stock_by(quantity)
    update!(stock: stock - quantity)
    broadcast_action_to(
      self,
      action: "reload",
      target: self,
      render: false
    )
  end
end
```

And we need to register to these events in the templates with `turbo_stream_from`:

```erb
<%# app/views/model/products/index.html.erb %>
<% @products.each do |product| %>
  <% item = @basket_items[product.id] %>
  <%= turbo_stream_from product %>
  <%= turbo_frame_tag product, src: card_product_path(product) %>
<% end %>
```

```erb
<%# app/views/model/products/show.html.erb %>
<%= turbo_frame_tag @product, data: {src: product_path(@product)} %>
  <%= turbo_stream_from @product %>
  <h1>@product.name</h1>
  <div>
    <p><%= @product.description %></p>
    <div>Origin: <%= @product.origin %></div>
    <div>stock: <%= @product.stock %></div>

    <% if @item&.stock_limit_reached? %>
      <div class="button-muted">no more</div>
    <% else %>
      <%= button_to "Add one", items_path, params: {product_id: @product.id}, method: :post %>
    <% end %>
  </div>
<% end %>
```

And here we are, that is - to me - the simplest and most efficient way to implement a "refresh a single frame" behavior. We could day-dream that this be part of Hotwire natively one day but in the mean time, this works great!
