---
layout: blogpost
title: From jQuery to Hotwire
subhead: Lessons of a successful migration
excerpt: Some lessons I learned successfully migrating Robin des Ferme's jQuery + Rails UJS Frontend to Hotwire, and a few tips & tricks if you are attempting the same feat.
categories: code
date: 2024-09-24 22:50:00 +0300
imgclass: robindesfermes
---

I joined [Robin des Fermes](https://www.robindesfermes.ch/) in January 2024, as sole developer, working 2.5 days per week with them. The main priorities they gave me were mostly Backend oriented. Frontend work wasn't a priority at that time, but soon enough there were features that would involve some Frontend work.

Robin des Fermes is a bootstrapped<sup id="sup-1"><a href="#footnotes-1">1</a></sup> company where the resources are scarce: 2.5 days of dev time per week, and we can't waste any money or time on non essentials. This definitely influenced some of the decisions I made when it comes to Frontend work.

Now this story is about a successful migration from the original code to Hotwire. So let's dive into how it happened.

## Initial State of the Frontend

The Frontend architecture at the time I joined was pretty much the classic "old" Rails default: some Rails UJS (the most famous of the UJS features is probably what you commonly know as `remote: true` in a form helper). And a bunch of jQuery snippets everywhere, not really centralized or organized. Plus a few JavaScript libraries imported to do some fancy things like a fancy Confirm Modal, a sliding menu, a carousel, stuff like that. Oh and also, Bootstrap (v4) was in there.

All in all, that's a pretty classic Rails Frontend from the pre-Hotwire era.

## Slow introduction of Hotwire

My mantra was: if I'm going to add any JavaScript code, I won't add any more jQuery or Rails UJS. I'll do *new* things with Hotwire (Stimulus & Turbo), while keeping all the *old* code around for existing features, for as long as it's not needing refactor. And then, whenever I get some time, or the old code gets in the way, I'll refactor it to migrate it to Hotwire. I did not have any timeline in mind. Just take it as it comes.

That lead me to introduce Stimulus in the app on February 7th 2024, in a PR where I needed simple Tab toggler, which I built with a Stimulus controller.

Stimulus is completely unobtrusive, so it's really safe to introduce it without worrying about old code still working. It is also very self-contained: you don't need Turbo to have Stimulus work, it just works fine as a standalone library. So go nuts with Stimulus.

### Stimulus tip for migrations: have a global controller

Getting a bit ahead of ourselves, but this is my only "Tip" that is exclusive to Stimulus, so I thought of giving it here while we are talking about it: when you start migrating old jQuery code to Stimulus controllers - so not when adding new behaviors to new views, but refactoring old ones - then you may find yourself in a bit of a complex quest:

A Stimulus controller is isolated to the element - and its children - where it is declared. For example if you have this bit of HTML:

```html
<button id="outside">I'm outside the stimulus controller</button>
<div data-controller="toggle" data-toggle-target="thisIsLegal">
  <span data-action="click->toggle#willWork"></span>
</div>
```

You can't target the button with `id="outside"` with the `toggle` controller, because the button lives outside of the tree of the element defining the controller.

But when you are migrating old code to Stimulus, it may be hard to identify the scope. A common jQuery pattern was to have a global document `DOMContentLoaded` event listener, and in the callback function, query some DOM element by class name (or id, data attribute; you get the point), and act onto other DOM elements.

That code could target _any_ part of the DOM and trigger action on _any other_ part of the DOM. There wasn't any scoping limit in any way.

So I found it helpful to have a `main` controller that I plugged onto the document `body` directly:

```html
<body data-controller="main">[everything]</body>
```

This way, I can migrate behavior to this controller little by little, without worrying about Stimulus controller scope, and by doing so, I start to see more clearly where the triggers are, where the actionable elements are, and I can then extract those actions to a specific Stimulus controller.

But the "just put stuff in main controller" intermediate step was really helpful to eliminate mental overhead.

Also, some libraries may not give you so many options on how to interact with them, they may themselves listen to the whole DOM (example: the [@mapbox/search-js-web](https://docs.mapbox.com/mapbox-search-js/api/web/autofill/) library has such a global listener for address autofill). In that case, you kinda _have_ to put this code in a global controller.

At the end of the migration, I still have a few actions in that global main controller: show & hide loading overlay (on the whole page), and that mapbox address autofill I just mentioned.

## Turbo is a different beast altogether

Contrary to Stimulus, Turbo is _very_ obtrusive and will get in the way real fast. Mostly when refactoring old views, but not only. So let's get into the main lessons I learned introducing Turbo in an existing project and eventually refactoring old views with to allow global Turbo Drive. And I'll add a few tips and tricks if you intend such a migration yourself.

### 1. Deactivate Turbo drive globally

First of all, this is not even a tip, it's a critical step if you aren't on a Greenfield project: turn Turbo Drive off globally.

Unless you want to migrate your whole Frontend at once (which you don't), then you definitely don't want Turbo Drive enabled until you are sure nothing is in its way (and lots will get in its way). I didn't find the docs very clear in that regards, here is how you do it:

In your entrypoint file (application.\[js\|ts\] most likely), add these lines right after importing turbo:

```javascript
require("@hotwired/turbo-rails"); // by default Turbo Drive is enabled globally
Turbo.session.drive = false; // Do not enable Turbo Drive globally
Turbo.session.history.stop(); // Disable history cache
Turbo.setFormMode("optin");
```

And now, whenever you want to use Turbo, you need the surrounding HTML element to have the `data-turbo="true"` attribute. You'll be able to remove all of those whenever you enable it globally but for now it's required:

```html
<div>No turbo here</div>
<div data-turbo="true">
  <a href="/something" data-turbo-frame="my-frame">I can target a Turbo Frame from here</a>
  <form data-turbo-stream="true" method="post" action="/stuff">
    I can also do form requests with Turbo Streams
  </form>
</div>
```

Note: there is an exception to this rule. Turbo frame elements do not need explicit `data-turbo="true"`. It's the only Turbo element which will automatically cause Turbo Drive features to become enabled for itself and all of its content. So this will work:

```html
<div>No turbo here</div>
<turbo-frame id="my-frame">I can use turbo there</turbo-frame>
```

Why do you need to deactivate Turbo Drive globally? Well Turbo in general (Drive, Frames, Streams) - but Turbo Drive in particular - is an extra obtrusive framework that gets in the way real fast.

The main issue I faced is having a lot of code being in a `DOMContentLoaded` event listener callback function. In fact, most of my JavaScript execution was following that pattern - as I said earlier, this used to be very common in jQuery times:

```javascript
document.addEventListener("DOMContentLoaded", (event) => {
  // all my code was here
});
```

Because Turbo swaps some parts (or all) of the HTML without reloading the page, then this callback function only gets called on the first page render, and on none of the other subsequent page visits. So if you have any code that queries the DOM (which, remember, most of it does, and then acts consequently), it will ignore any HTML snippet replaced after the first page visit.

So you'll have to get rid of all the code that gets executed in a `DOMContentLoaded` event callback before Turbo Drive can be turned on globally.

The secondary reason is, Turbo is not compatible with Rails UJS (I may generalize - but mostly just consider it not compatible). So any `method: :delete` in a form helper needs to be replaced with `data: {turbo_method: :delete}` for example. Until this is all migrated, do not turn Turbo Drive on globally or all of these requests will break.

In my case, this global enabling is the very last thing I did to complete the migration, and I was very nervous about doing so, always fearing I missed something<sup id="sup-2"><a href="#footnotes-2">2</a></sup>.

### 2. Turbo is mostly fine if you add it to new views

If you are adding new views, you can use Turbo Frames into them, and have your controller respond to request with Turbo Streams, you won't get many surprises (™famous last words).

The main thing to beware of is, there is [no default way](https://github.com/hotwired/turbo-rails/pull/367) to redirect the request outside of the current frame.

I solved this following [the suggestion of a contributer](https://github.com/hotwired/turbo-rails/pull/367#issuecomment-1934729149) in that very github thread; In my JavaScript, I have this custom action defined after importing hotwire:

```javascript
Turbo.StreamActions.redirect = function () {
  Turbo.visit(this.target);
};
```

And in my application controller, I have this method:

```ruby
def redirect_turbo_to(path)
  respond_to do |format|
    format.html { redirect_to path }
    format.turbo_stream do
      render turbo_stream: turbo_stream.action(:redirect, path)
    end
  end
end
```

And now in my controllers, I can do `redirect_turbo_to my_path` and it will do the "correct" thing whether it's a Turbo Stream request or a HTML request.

Another gotcha is, every "failed" request must have a `422` response status. This is documented [here](https://turbo.hotwired.dev/handbook/drive#redirecting-after-a-form-submission) but again, to me that wasn't so straightforward to find.

Here is what I mean, say you have a controller like this, which is pretty standard:

```ruby
def create
  @resource = Resource.new(permitted_params)

  if @resource.save
    redirect_turbo_to resources_path
  else
    render :new
  end
end
```

This will not show the errors of `@resource` in your form on `render :new`. Instead the request must respond with the `422` status code:

```ruby
def create
  @resource = Resource.new(permitted_params)

  if @resource.save
    redirect_turbo_to resources_path
  else
    render :new, status: :unprocessable_entity
  end
end
```

### 3. Turbo really only works well in combination with Stimulus

Now, let's imagine that you have an interaction where you want to use a Turbo Frame, maybe respond with some Turbo Streams, and within that Turbo Frame, you have some JavaScript interaction. For example maybe you want to open a modal when a user clicks a button.

The JavaScript code to open this modal **can not** be initialized within a global `DOMContentLoaded` event listener callback function. Otherwise no modal will ever open after the first Turbo render.

Here is an example, say you have this code:

```html
<turbo-frame id="my-frame">
  <!-- Imagine here a form that will update the frame with the response of its submission -->

  <!-- and then after the form, we also have this extra bit: -->
  <button class="modal-trigger">Open Modal!</button>
  <div class="modal"><span>My modal content</span></div>
</turbo-frame>
```
```javascript
document.addEventListener("DOMContentLoaded", (event) => {
  document.querySelectorAll(".modal-trigger").forEach((el) => {
    el.addEventListener("click", () => {
      const modal = document.querySelector(".modal")
      if (modal) modal.classList.add("show")
    }
  })
});
```

Well this JavaScript code will run only once, on page load (let's imagine this page is the first you visit, for the sake of simplicity). Then you submit the form, this updates the frame and wooops! Now the button doesn't work anymore, it won't open the modal on click.
Well now we understand how that makes sense. So what are your alternatives?

1) You can inline the onclick function like this:

```html
<button class="modal-trigger" onclick="document.querySelector('.modal')?.classList?.add('show')">Open Modal!</button>
```
Well I guess you know not to do that<sup id="sup-3"><a href="#footnotes-3">3</a></sup>.

2) You can replace the `DOMContentLoaded` event listener with a `turbo:load` event listener, or is it `turbo:before-stream-render`? Again, that will quickly become a nightmare, don't do that (also, depending on what you want to do, it won't even work).

3) You can add a Stimulus controller. That's basically what Hotwire wants you to do. Then it becomes cleaner:

```html
<turbo-frame id="my-frame" data-controller="modal">
  <button class="modal-trigger" data-action="click->modal#open">Open Modal!</button>
  <div class="modal" data-modal-target="body"><span>My modal content</span></div>
</turbo-frame>
```

```javascript
import { Controller } from "@hotwired/stimulus";
// Connects to data-controller="modal"
export default class extends Controller {
  static targets = ["body"]
  open() {
    this.bodyTarget.classList.add("show")
  }
}
```

So yeah, if you planned to use Turbo without Stimulus... Well good luck and tell me how you did it, I'm genuinely curious.

### 4. Beware of the links

So far I only gave examples of adding Turbo to new views. But its obtrusive nature really gets obvious when you start to add Turbo Frames to existing code. As in, there is a block of HTML that you wrap with a Turbo Frame because you want to do Turbo things in it.

Well, you already know about the `DOMContentLoaded` part: if you have any JavaScript code listening to some element via their class name (id, etc) in that block of HTML, you'll have to migrate this JavaScript code to a Stimulus controller. This may or may not be straightforward to identify.

But there is another thing I want to point out: by default, a link within a turbo frame targets _the frame itself_. What that means is, if you have some code like this:

```html
<!-- some HTML before the frame, for example: -->
<img src="/hello.jpg" alt="hello" />

<turbo-frame id="my-frame">
  <a href="/resources/3">Show resource</a>
</turbo-frame>

<!-- more HTML here, for example: -->
<img src="/bye.jpg" alt="bye" />
```

Turbo expects `GET /resources/3` to respond with some HTML that will look like this:

```html
<turbo-frame id="my-frame">
  <span>This is resource 3</span>
</turbo-frame>
```

Then it will swap the original frame with that payload, and the final result is:

```html
<!-- some HTML before the frame, for example: -->
<img src="/hello.jpg" alt="hello" />

<turbo-frame id="my-frame">
  <span>This is resource 3</span>
</turbo-frame>

<!-- more HTML here, for example: -->
<img src="/bye.jpg" alt="bye" />
```

If the rendered HTML does not contain `<turbo-frame id="my-frame">[...]</turbo-frame>` you will get the infamous `Content missing` error.

So if the `GET /resources/3` request returns a full HTML page - which is what your app was doing before you added Turbo, that's still probably what it does after you added it - then you want to break out of the frame. There are several ways to do it.

One option is to have a [global meta tag](https://turbo.hotwired.dev/handbook/frames#%E2%80%9Cbreaking-out%E2%80%9D-from-a-frame). That's not very convenient if you want to be more granular.

Another option is, you can add a `target="_top"` to the frame itself like this:

```html
<turbo-frame id="my-frame" target="_top">
  <a href="/resources/3">Show resource</a>
</turbo-frame>
```

This will work, now the link will target the `_top` frame. But beware: if you have any Turbo Streams request made from within the frame, they also target the `_top` frame by default. So now you need to update those if they were intended to target the frame:

```html
<turbo-frame id="my-frame" target="_top">
  <a href="/resources/3">Show resource</a>
  <form action="/resources" method="post" data-turbo-stream="true" data-turbo-frame="my-frame">
    Some form component
  </form>
</turbo-frame>
```

What you gain by not having to update your links, you lose by having to update your forms. Depending on your use-case it's an interesting trade-off. Note that in the case of form response targeting the `_top` frame when they intended to target a specific frame, you will not get any `"Content Missing"` error, it will just look like nothing happened.

The last option is, you can go update all the links within the frame - and that's the solution I would recommend - which might be a pain depending on the complexity of the view: you may have multiple partials, maybe some of them are conditional, etc.

The safest way is to add `data-turbo="false"` on the links: this escapes from Turbo Drive and will render the HTML response as a full page load. This is the safest, although on the long run, you will want Turbo enabled, so that's not awesome.

The alternative is to add `data-turbo-frame="_top"` to every link: this will target the `_top` frame (which is added by Turbo itself) and swap it with the HTML response, which will look like a full page render to the user. But it is actually a full page swap, so now you need to make sure that this view (`GET resources/3`) does not contain any HTML element that is listened on from a `DOMContentLoaded` event listener callback function. Which may or may not be easy to identify and refactor.

### 5. Beware of the libraries (in particular those you don't expect - like Bootstrap)

As I mentioned before, libraries you import into your code may be designed to run code in a `DOMContentLoaded` event listener callback.

If you want to use them with Turbo, that's gonna be tricky (or impossible?). I think that probably only concerns old jQuery based libraries. I don't know. In my case, I managed to get rid of all of those offending libraries and either replace them with alternative libraries, or with home made JavaScript & CSS.

One case that I found tricky though, is Bootstrap<sup id="sup-4"><a href="#footnotes-4">4</a></sup>.

As long as you only use the CSS parts of Bootstrap (v4), you are safe. But you may be using some of its JavaScript features, like tooltips, popovers, modals, or carousels.

And if you trigger them with `data` attributes in your HTML, you'll have to replace those with explicit JavaScript invocation (Again, because their data attribute invocation are managed via a `DOMContentLoaded` event listener callback function).

For example, I replaced every `data-toggle="tooltip"` with a `data-controller="tooltip"` Stimulus controller. This is what the controller does:

```javascript
import { Controller } from "@hotwired/stimulus";
import { Tooltip } from "bootstrap";

// Connects to data-controller="tooltip"
export default class extends Controller {
  connect() {
    this.tooltip = new Tooltip(this.element);
  }

  disconnect() {
    this.tooltip.dispose();
  }
}
```

This is pretty straightforward and Bootstrap (v4) provide the JavaScript APIs to do that. What is tricky is to identify what you use or not. In my case, I almost missed 2 `data-ride="carousel"` for example.

The modal is more tricky to migrate, and if you are in the same case as me, [this blog post](https://www.hotrails.dev/articles/rails-modals-with-hotwire) helped a lot to use the Bootstrap modal together with Stimulus (modal controller) and Turbo (via events triggering controller actions).

## Migration complete

As you may guess, I learned these lessons sometimes the hard way. I did overlook quite a few time the effect of adding a Turbo Frame around an existing bit of code, only to later realized that a sliding menu wasn't working anymore. Or that a link was incorrectly targeting the surrounding frame instead of the `_top` frame.

But in the end, I had enough time & good reason to refactor all the legacy code and migrate it to Hotwire.

Eventually, on September 24th 2024, 230 days after introducing Hotwire (Stimulus) to the project, I completed the migration by enabling Turbo Drive globally 🎉.

As you can see, that was yesterday, so I may still learn lessons from this migration in the coming days 😅

Overall, I'm very happy with this migration, and to now be solely<sup id="sup-5"><a href="#footnotes-5">5</a></sup> relying on Hotwire for the Frontend. This gives me quite some confidence for future work in this application.

---

[<span id="footnotes-1">1</span>] <a href="#sup-1">↑</a> Still true as of today (September 24th 2024), but not for much longer: Robin des Fermes is currently doing its first community based fundraising. That being said, it will likely not change the very agile nature of the work & overall mindset of the company.

[<span id="footnotes-2">2</span>] <a href="#sup-2">↑</a> Spoiler: I did miss something, one Rails UJS usage, and had to push a Hotfix 30 minutes after enabling Turbo globally. Good times

[<span id="footnotes-3">3</span>] <a href="#sup-3">↑</a> Except in some very rare case - `onchange="this.form.requestSubmit()"` is one example that comes to my mind - and even then I wouldn't advocate for inline JavaScript.

[<span id="footnotes-4">4</span>] <a href="#sup-4">↑</a> Or any other CSS framework that actually does more than CSS, like Semantic UI for example

[<span id="footnotes-5">5</span>] <a href="#sup-5">↑</a> Well of course that would be too easy. I still have the whole admin area - built with activeadmin - relying on legacy Rails Frontend (jQuery & Rails UJS). But at least the main app has declared its freedom 🏴󠁧󠁢󠁳󠁣󠁴󠁿
