---
layout: blogpost
title: Journey of a Trailblazer (2)
subhead: A road to refactoring
categories: code
imgclass: trail
redditlink: "https://www.reddit.com/r/ruby/comments/4z5yqv/journey_of_a_trailblazer_refactoring_a_real_rails/"
---

*This blog post is the part 2 of my refactoring experience with the [Trailblazer](http://trailblazer.to/) architecture. If you haven't already, you should read [part 1]({% post_url 2016-08-21-journey-of-a-trailblazer-part-1 %}) first.*

In part 1, I showed how I refactored the Create operation of a Competition with an Operation and a Form object. In this post, I will show how I refactored the rendering of a form object, and how I refactored the Update operation of a Competition.

## Rendering an empty form

In part 1 we refactored the Create operation of a Competition. Now is the time to render the form in `CompetitionsController#new`.

In this step, I want to refactor:

- The `new` method of `CompetitionsController`
- Populate the html form with a form object instead of a Competition object.

I don't want to refactor anything else. I want to keep my partials (no refactor with Cells yet), and keep pundit for authorization in the controller.

Here is the code I want to refactor:

```ruby
# app/controllers/competitions_controller.rb
def new
  # I want to render a @form object instead of @competition and @tracks
  @competition = current_user.creations.new
  authorize @competition, :create?

  @competition.build_start_city
  @competition.build_end_city

  # The rendered form contains one track by default, the user can add more 
  # if he wants, but must keep at least one.
  @competition.tracks.build
  @competition.tracks.last.build_start_city
  @competition.tracks.last.build_end_city
  @tracks = @competition.tracks
end
```

```erb
<!-- app/views/competitions/new.html.erb -->
<div class="container padded-mini">
  <div class="row">
    <div class="col-sm-8 col-sm-offset-2">
      <h3><%= t('.title') %></h3>

      <%= render 'form', competition: @competition, tracks: @tracks %>
    </div>
  </div>
</div>
```

The form partial is very big so I won't paste it here. But it is available on [github](https://github.com/NoryDev/somewherexpress/blob/1a85cbba55e57e5a2a98373d1779c314f9f3bd5c/app/views/competitions/_form.html.erb).

### The controller method and general view

In the controller method, we can reuse the form that we made for `Competition::Create` as explained in the Trailblazer book p. 55:

```ruby
# app/controllers/competitions_controller.rb
def new
  authorize Competition, :create?

  @form = form Competition::Create
end
```

Then we can use this `@form` object in the view.

```erb
<!-- app/views/competitions/new.html.erb -->
<h3><%= t('.title') %></h3>

<%= render 'form', competition: @form %>
```

we should rename the `:competition` to `:form`, but this would break the `edit` view, who uses the same form. But as we will soon see, it's going to be broken anyway. As a temporary solution, we could have two different form partials for `new` and `edit`. But we will rather refactor both `new` and `edit` to avoid this annoyance.

For now, let's see what breaks in the form partial. First, we don't pass a `:tracks` attribute anymore, so let's change this:

```erb
<!-- app/views/competitions/_form.html.erb -->
<%= f.simple_fields_for :tracks do |t| %>
<!-- was f.simple_fields_for :tracks, tracks do |t| -->
<!-- this will break on edit -->
```

Now, in this partial, we also have the form to destroy a competition. It breaks because it's wrapped with an authorization condition that requires an instance of Competition (and not a form object).

This destroy form will never be rendered in the case of en empty form. It has nothing to do in this partial, it should be in it's own partial and be rendered only in case of updating a competition. So I'm just going to extract it in it's own partial and not render it in the main form partial ([this commit](https://github.com/NoryDev/somewherexpress/commit/d16768f51ef4de7cf306289079c4dea8e7dd80d3) shows what I did).

This small refactor might not seem very relevant, but I included it in this blog post, because for me, it's a example of how Trailblazer helps me structure my code in a meaningful way. The fact that I have a form object instead of a competition object forces me to separate concerns, structure my code, and clean my mess. One more 👍 for you Trailblazer.

### Building the nested objects

Now, the page renders. But it has no nested forms. That makes sense; in the old controller behavior we had some manual building of cities, one track, and this track's cities:

```ruby
# app/controllers/competitions_controller.rb
def new
  # [...]
  # We removed that part:
  @competition.build_start_city
  @competition.build_end_city

  @competition.tracks.build
  @competition.tracks.last.build_start_city
  @competition.tracks.last.build_end_city
end
```

In the `Competition::Contract::Create`, Reform allows us to build nested objects with the `:prepopulator` attribute.

Now, the tricky part, that took me a long time and a lot of struggle to understand, is the differences between `:populate_if_empty` and `:prepopulator`. Because Form objects works in two directions: 

1. **Incoming**: Deserialize and validate incoming data.
2. **Outgoing**: Render outgoing data from the Database in a html form.

For nested forms, `:populate_if_empty` is used in the *Incoming* direction to populate for validation, and `:prepopulator` in the *Outgoing* direction to populate a html form. It was very confusing to me because they are both called 'populate'-something and I still don't find the naming very intuitive.

In the Trailblazer book, p. 94, the paragraph called “Prepopulation vs. Validation Population” is key to understand this difference.

Let's see how to use `:prepopulator` with `start_city` and `end_city`. Like with `:populate_if_empty`, we can pass a method to `:prepopulator`. In the case of a html form to create a new competition, we want a new empty city. Here is the syntax: 

```ruby
# app/concepts/competition/contract.rb
class Competition < ActiveRecord::Base
  module Contract
    class Create < Reform::Form
      # [...]
      property :start_city, prepopulator: :prepopulate_start_city!,
                            populate_if_empty: :populate_city!,
                            form: City::Form

      property :end_city, prepopulator: :prepopulate_end_city!,
                          populate_if_empty: :populate_city!,
                          form: City::Form
      # [...]
      private

        def prepopulate_start_city!(_options)
          self.start_city = City.new
        end

        def prepopulate_end_city!(_options)
          self.end_city = City.new
        end

        # [...]
    end
  end
end
```

Now our form renders city but not tracks. We can use the same `:prepopulator` method on a collection. In this case, we should also build a `start_city` and a `end_city` for the newly prepopulated track:

```ruby
# app/concepts/competition/contract.rb
class Competition < ActiveRecord::Base
  module Contract
    class Create < Reform::Form
      # [...]
      collection :tracks, prepopulator: :prepopulate_tracks!,
                          populate_if_empty: :populate_track! do
        # [...]
      end

      private

        def prepopulate_tracks!(_options)
          track = Track.new
          track.build_start_city
          track.build_end_city
          tracks << track
        end
        # [...]
    end
  end
end
```

### Helpers in form

Now if we try to render our form... It breaks! I'm using a helper in my form, `t.object.new_record?` to figure if the Track is a new one or an existing one. That is useful when the user removes a track from the form, to know if we can just hide the track form or if we need to make a delete request to destroy it from the database.

Now that I have a form object instead of a competition object, the form object doesn't understand the `new_record?` method. We can find a solution to this problem in the Trailblazer book p. 144. Let's update the collection:

```ruby
# app/concepts/competition/contract.rb
class Competition < ActiveRecord::Base
  module Contract
    class Create < Reform::Form
      # [...]
      collection :tracks, prepopulator: :prepopulate_tracks!,
                          populate_if_empty: :populate_track! do
                          # See p. 145 why I don't need inherit: true, not
                          # inheriting anything here
        def new_record?
          !model.persisted?
        end

        # [...]
      end
    end
  end
end
```

Our form is complete and renders everything we want. However, the field for `:description` is no more a textarea, but a text input. Also, the required fields are wrong. We won't find anything about this in the book, but in [this part](http://trailblazer.to/gems/reform/rails.html#simple-form) of the Reform documentation, we can see that we can add a module for simple_form to our contract. This will solve these 2 issues:

```ruby
# app/concepts/competition/contract.rb
class Competition < ActiveRecord::Base
  module Contract
    class Create < Reform::Form
      include ActiveModel::ModelReflections
      # [...]
    end
  end
end
```

Side note: before I start these series of refactoring, I had a pretty ugly hack: in controller methods, I would manually add an empty track to my competition object, and then remove it with javascript in the view. Because my code is clean and nice now, I did not feel like re-implementing this hack in controller methods, so I fixed the javascript part that required me to add this empty track (in [this commit](https://github.com/NoryDev/somewherexpress/commit/5c151dc5167152905c0c78555dab9b96977497aa)). That's another example of Trailblazer forcing me to clean my mess.

## Rendering a form to edit an existing object

The `edit` method is now broken, because we modified the form partial. Let's refactor this part. It's very similar to the `new` method. Here is the code to refactor:

```ruby
# app/controllers/competitions_controller.rb
class CompetitionsController < ApplicationController
  before_action :set_competition, only: [:show, :edit, :update, :destroy]
  
  def edit
    authorize @competition, :update?

    @tracks = @competition.tracks.order(:start_time, :created_at)

    # This under was part of the "add an empty track" hack that we fixed.
    # Should be removed.
    track = @competition.tracks.build
    track.build_start_city
    track.build_end_city
    @tracks << track
  end

  private

    # I don't want to refactor this before_action method for now
    def set_competition
      @competition = Competition.find(params[:id])
    end
end
```

```erb
<!-- views/competitions/edit.html.erb -->
<div class="container padded-mini">
  <div class="row">
    <div class="col-sm-8 col-sm-offset-2">
      <h3><%= t('.title') %></h3>

      <%= render 'form', competition: @competition, tracks: @tracks %>
    </div>
  </div>
</div>
```

The view should look like this:

```erb
<!-- views/competitions/edit.html.erb -->
<h3><%= t('.title') %></h3>

<%= render 'form', competition: @form %>
```

To pass a form to the view, we can manually call our existing Create form. We should in fact pass a Update form. But Update form is going to be exactly the same as Create form, with the exception of prepopulation (`:prepopulator` attributes). There is no need for prepopulation in the case of an edit form.

An interesting fact:

In the `new` method, we were attributing the form part of the `Competition::Create` Operation like this:

```ruby
def new
  @form = form Competition::Create
end
```

This actually executes two operations:

1. Create a new empty form object
2. call the `prepopulate!` method on this form object. It's a shortcut for:

```ruby
def new
  @form = Competition::Create::Contract.new
  @form.prepopulate!
end
```

As we don't want prepopulation in the case of `edit`, we can simply do:

```ruby
def edit
  authorize @competition, :update?

  @form = Competition::Contract::Create.new(@competition)
end
```

I find this a bit hackish, and I would prefer to have a `Contract::Update`. But we will do that when we refactor the update operation.

You remember the form to destroy a competition? We need it here, and this form will need a `@competition` object:

```erb
<!-- views/competitions/edit.html.erb -->
<h3><%= t('.title') %></h3>

<%= render 'form', competition: @form %>
<%= render 'destroy_form', competition: @competition %>
```

## Refactoring the Update operation

The Update operation is very similar to the Create operation. As we just saw, it uses almost the same contract. The callback is different though.

This is the code to refactor:

```ruby
# app/controllers/competitions_controller.rb
def update
  authorize @competition

  updater = Competitions::Update.new(@competition, params).call
  @competition = updater.competition
  @tracks = updater.updated_tracks

  if @competition.valid? && @tracks.map(&:valid?).all?
    if @competition.just_published?
      send_new_competition_emails
    elsif @competition.published? && !@competition.finished? && @competition.enough_changes?
      send_competition_edited_emails
    end

    redirect_to @competition
  else
    # This is again for the now deprecated javascript hack
    track = Track.new(end_city: City.new, start_city: City.new)
    @tracks << track

    render :edit
  end
end

private

  # send_new_competition_emails, same method as for create

  def send_competition_edited_emails
    User.want_email_for_competition_edited(@competition).each do |user|
      UserMailer.as_user_competition_edited(user.id, @competition.id).deliver_later
    end
  end
```

Let's start with the `Competition::Update` Operation. It can inherit from `Competition::Create`, as it basically does the same. The test for this operation would look like this:

```ruby
RSpec.describe Competition::Update do
  let!(:user) { FactoryGirl.create(:user) }

  it "updates a competition" do
    # Use Competition::Create as factory:
    competition = Competition::Create
                  .call(competition: {
                          name: "new competition",
                          published: "1",
                          start_date: 2.weeks.from_now.to_s,
                          end_date: 3.weeks.from_now.to_s,
                          start_registration: Time.current,
                          finished: false,
                          start_city: { name: "Yverdon, CH",
                                        locality: "Yverdon-Les-Bains",
                                        country_short: "CH" },
                          end_city: { name: "Munich, DE",
                                      locality: "Munich",
                                      country_short: "DE" },
                          tracks: [{ start_time: 16.days.from_now.to_s,
                                     start_city: { name: "Yverdon, CH",
                                                   locality: "Yverdon-Les-Bains",
                                                   country_short: "CH" },
                                     end_city: { name: "Munich, DE",
                                                 locality: "Munich",
                                                 country_short: "DE" } }]
                        },
                        current_user: user)
                  .model

    Competition::Update.call(id: competition.id,
                             competition: { name: "updated name" })

    competition.reload
    expect(competition.name).to eq("updated name")
  end
end
```

```ruby
# app/concepts/competition/operation.rb
class Competition < ActiveRecord::Base
  class Update < Create
    action :update

    # We will implement a contract just for update
    contract Contract::Update

    def process(params)
      validate(params[:competition]) do |f|
        f.save

        # let's directly put the email sending callback in there:
        if model.just_published?
          send_new_competition_emails
        elsif model.published? && !model.finished? && model.enough_changes?
          send_competition_edited_emails
        end
      end
    end

    private

      # send_new_competition_emails is inherited from Create and not 
      # overridden.

      def send_competition_edited_emails
        User.want_email_for_competition_edited(model).each do |user|
          UserMailer.as_user_competition_edited(user.id, model.id).deliver_later
        end
      end
  end
end
```

And the contract can also inherit from Create and override only the necessary parts:

```ruby
# app/concepts/competition/contract.rb
class Competition < ActiveRecord::Base
  module Contract
    class Update < Create
      private

        # As said previously, no prepopulation for Update:
        def prepopulate_tracks!(_options)
        end

        def prepopulate_start_city!(_options)
        end

        def prepopulate_end_city!(_options)
        end
    end
  end
end
```

The test is passing. It's very nice to use `Competition::Create` instead of a factory, but with that many parameters, I would still like to have some kind of factory, in order to avoid entering fake data for my params every time I need a competition. In that regard, FactoryGirl is very cool. I like the ability to do `FactoryGirl.create(:user)` and it creates a user with fake data. I might search how to do that with Operations at some point, but not now.

Now we can implement the Update operation in the controller, again similar to the `create` method:

```ruby
# app/controllers/competition_controller.rb
def update
  authorize @competition

  operation = run Competition::Update,
                  params: params.merge(current_user: current_user) do |op|
    return redirect_to op.model
  end

  @form = operation.contract
  render action: :edit
end
```

Now we can modify the `edit` method to use our newly created operation:

```ruby
def edit
  authorize @competition, :update?

  @form = form Competition::Update
end
```

Here we are! A nice and clean controller class, every part in it's own relevant class. We can do a little bit of cleaning:

- In the form partial, rename the `:competition` attribute to `:form` to avoid confusion.
- Remove the old `Competitions::Update` service ([the old one](https://github.com/NoryDev/somewherexpress/commit/19b1b3406975540dbd6e344563f257791a86001b), with 's' in Competition**s**) that is not used anymore.
- Remove validations in Competition, Track and City models.
- Remove `accepts_nested_attributes_for` in Competition and Track models.

Well actually, I can't remove validations and `accepts_nested_attributes_for` from models because I use [activeadmin](https://github.com/activeadmin/activeadmin) who relies on them. I'm going to remove anyway: I'm the only admin on this app, so it will only affect me when it's not working.

After this refactoring, I think I can ditch activeadmin's create/update functionalities and just keep it for Read and Destroy. It's going to be much easier to create/update competitions from the terminal with my new Operations. Another 👍 for you, Trailblazer.

The final code of the entire refactor (part 1 plus part 2) is [here](https://github.com/NoryDev/somewherexpress/tree/767fa2cd85af9cc19159a160a0f9e030e7afe6ec).

## Conclusion

That's it for now. I could refactor more, and I will, but it's a story for another time. Here are a few conclusion points that I draw from this experience:

- Trailblazer promised that I could refactor one part after another, and it proved to be true. I give it 👍👍
- Trailblazer promised to provide structure and conventions, and I like the choices made by Operations (especially the chaining possibilities) and Reform in that regard. I give it 👍👍
- Reform is an awesome form object library, it was super nice to use in the case of my unconventional City attribution. But I still think the `:populate_if_empty` and `:prepopulator` attribute names are confusing. Still, I give it 👍👍
- Trailblazer promised that I could use Operations to replace factories. It works fine, but I still want a factory to seed my Operation with fake params. I give it 👍
- The book is very helpful and detailed. However, it was a little bit challenging to use in the case of a refactor, had to jump between chapters. When something is not in the book (like the `simple_from` stuff), it can be tricky to find solutions in the documentation. I still give a 👍 to the book because it's a nice tool.

As a final comment about this overall experience, I think Trailblazer should be treated like a framework on it's own. I was barely less struggling and confused than when I learned Rails for the first time. There is a lot of conventions going on, and in a way, it's own magic under the hood. I think it works well with Rails, it kind of extends Rails CoC principles.

This need to be known if you plan to have a full-Trailblazer app: your developers, and the developers you will hire will have to learn it, even if they already know Rails. And it might be a learn-a-new-framework kind of experience for some of them.

That being said, I'm looking forward to see what else I can do with Trailblazer. My journey is not over yet.
