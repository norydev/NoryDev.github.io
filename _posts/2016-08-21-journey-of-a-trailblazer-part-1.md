---
layout: blogpost
title: Journey of a Trailblazer (1)
subhead: A road to refactoring
categories: code
imgclass: trail
---

I decided to refactor one of my Rails side projects with the [Trailblazer](http://trailblazer.to/) architecture. Why? The short version is, I'm looking for some standards and conventions to refactor my [day job](http://www.pisano.co)'s Rails API application. After reading a bit about Trailblazer and what it offers, I decided to try it on one of my side projects. I would refactor it to learn about the architecture and then decide if Trailblazer is a good fit for my professional application.

As I was doing the first batch of refactoring, I thought that my experience in struggling to understand how it works could benefit others. That's why I'm writing a blog post. As a matter of fact, as I was writing this blog post I discovered and learned some more stuff that did not occur to me when I was refactoring.

### The App

The app I'm going to refactor is [somewherexpress](https://github.com/NoryDev/somewherexpress), which allows my friends and I to organize our own hitchhiking competitions and showcase the results. You can see the result [here](https://www.somewherexpress.com).

It's a rather standard, small Rails 4.2 application: it has 8 ActiveRecord models + their relative controllers and views. Mostly CRUD actions with a few tricks, callbacks, nested forms with [simple_form](https://github.com/plataformatec/simple_form), [devise](https://github.com/plataformatec/devise), [pundit](https://github.com/elabs/pundit), transactional emails. No externally open API.

It is open source, so you can follow my struggles in the [trailblazer branch](https://github.com/NoryDev/somewherexpress/tree/trailblazer).

### Trailblazer

The Trailblazer book can be bought on [leanpub](https://leanpub.com/trailblazer). In this post, I will make references to the pdf version of the book (for page numbers).

The two introduction chapters are very promising:

> Trailblazer was created while reworking Rails code and does absolutely not require a green field with grazing unicorns and rainbows. It is designed to help you restructuring existing monolithic apps that are out of control.
> 
> <cite>&mdash; Trailblazer, p. 7</cite>

➡ This is essential

> Conventional factories in tests create redundant code that never produces the exact same application state as in production - a source of many bugs. While the tests might run fine production crashes because production doesn’t use factories. In Trailblazer factories get superseded by using operations.
> 
> <cite>&mdash; Trailblazer, p. 7</cite>

➡ I don't have too many problems with factories but that would be a good perk

> Instead of leaving it up to the programmer how to design their “service object”, what interface to expose, how to structure the services, Trailblazer’s Operation gives you a well-defined abstraction layer for all kinds of business logic.
> 
> <cite>&mdash; Trailblazer, p. 17</cite>

➡ Great, I don't want to make decisions, give me some conventions

> While rendering documents is provided with fantastic implementations, Rails underestimates the complexity of deserializing documents manually. Many developers got burned when “quickly populating” a resource model from a hash. Representers make you think in documents, objects, and their transformations - which is what APIs are all about.
> 
> <cite>&mdash; Trailblazer, p. 30</cite>

➡ That could solve major struggles in my day job application

## Refactoring the Create operation

The book recommends to start (or start refactor) with the most important business action. I'm not skeptical about this advice in the case of a refactor. In a sense, I get it, it's your core business, you want it to work well. But it's likely the most complex part of your code base. So starting by refactoring a huge method might prove challenging when you don't know the new architecture. I'm going to carry on with this advice anyway. So let's refactor The Competition creation process of somewherexpress. Here is the code I want to refactor:

```ruby
# app/models/competition.rb
class Competition < ActiveRecord::Base
  has_many :tracks, dependent: :destroy
  # I want to get rid of this accepts_nested_attributes_for
  accepts_nested_attributes_for :tracks, allow_destroy: true

  belongs_to :start_city, class_name: "City", foreign_key: "start_city_id"
  belongs_to :end_city, class_name: "City", foreign_key: "end_city_id"

  has_many :tracks_start_cities, through: :tracks, source: :start_city
  has_many :tracks_end_cities, through: :tracks, source: :end_city

  # I want to get rid of these accepts_nested_attributes_for
  accepts_nested_attributes_for :start_city
  accepts_nested_attributes_for :end_city

  belongs_to :author, class_name: "User"

  # These validations should not be in the model
  validates :name, presence: true
  validates :start_registration, :start_city, :end_city,
            :start_date, :end_date, presence: { if: :published? }
  #[...]
end
```

```ruby
# app/models/track.rb
class Track < ActiveRecord::Base
  belongs_to :competition

  belongs_to :start_city, class_name: "City", foreign_key: "start_city_id"
  belongs_to :end_city, class_name: "City", foreign_key: "end_city_id"

  # I want to get rid of everything below this
  accepts_nested_attributes_for :start_city
  accepts_nested_attributes_for :end_city

  validates :start_city, :end_city, :start_time, presence: true
  #[...]
end
```

```ruby
# app/controllers/competitions_controller.rb
def create
  @competition = current_user.creations.new
  authorize @competition

  # I want to get rid of Competitions::Update, which is a big mess.
  # it's a kind of custom form object to map Cities based on their locality
  # attribute and not their id. The strong parameters lie in this class
  updater = Competitions::Update.new(@competition, params).call
  @competition = updater.competition
  @tracks = updater.updated_tracks

  if @competition.valid? && @tracks.map(&:valid?).all?
    send_new_competition_emails if @competition.published?

    redirect_to @competition
  else
    # This is a hack I also want to get rid of: the rendered form requires an
    # empty track that is removed on load with javascript
    track = Track.new(end_city: City.new, start_city: City.new)
    @tracks << track

    render :new
  end
end

private

  # This method should not be in the controller
  def send_new_competition_emails
    User.want_email_for_new_competition.each do |user|
      UserMailer.as_user_new_competition(user.id, @competition.id).deliver_later
    end
  end
```

This piece of code has all these usual suspects, plus some perks:

- conditional validations
- `accepts_nested_attributes_for` join model
- callbacks
- a custom service (doing the job of a form object) because I don't want the default Rails behavior on create/update a join model (City).
- some rubbish decoration to comply with my rendered form

I want to check if I can refactor only one part at a time. This is an important factor for me. So as a first step I want to refactor:

- the `create` method of `CompetitionsController`
- use a form object to deserialize and validate the data to create a competition.

That's it. I want to keep devise for authentication, pundit for authorization, keep my callbacks, and render my html views. And this is where "just following" the book made me struggle a lot. The book is structured to demonstrate how to create an application from scratch using all the Trailblazer classes. So if we want to refactor just one part, we will have to jump entire sections.

Let's get started

### Trailblazer::Operation & Reform::Form

The **Operation** is the core service object in Trailblazer. An operation orchestrates all business logic between the controller dispatch and the persistence layer.

If we put aside authorization for now, we want to modify `CompetitionsController#create` to look like this:

```ruby
# app/controllers/competitions_controller.rb
def create
  run Competition::Create do |op|
    return redirect_to op.model
  end

  render action: :new
end
```

As explained in the Trailblazer book p. 49, if there is no exception raised when the operation is ran, the given block will be executed, otherwise it won't. I like this way of dealing with errors. It allows nesting services without the need of many nested conditions. Caution though, if you call your operation with the call syntax, exceptions won't be rescued.

So the first thing we need is a new file for this `Competition::Create` operation. I'm going to immediately integrate the **Contract** part - the form object layer that uses [Reform](http://trailblazer.to/gems/reform/), another Trailblazer gem - in the operation.

The form object is the most interesting part of this operation, so let's set aside authorization and the callback (send emails) for now, and concentrate on the contract. If you don't have nested forms, the operation + contract is quite straight forward following the Trailblazer book (chapter 3):

```ruby
RSpec.describe Competition::Create do
  it "creates an unpublished competition" do
    competition = Competition::Create
                    .(competition: { name: "new competition" })
                    .model

    expect(competition).to be_persisted
    expect(competition.name).to eq "new competition"
  end

  it "does not create an unpublished competition without name" do
    expect {
      Competition::Create.(competition: { name: "" })
    }.to raise_error Trailblazer::Operation::InvalidContract
  end
end
```

```ruby
# app/concepts/competition/operation.rb
class Competition < ActiveRecord::Base
  class Create < Trailblazer::Operation
    include Model
    model Competition, :create

    # The contract is extracted into a new file, it's quickly going to be big.
    contract Contract::Create

    def process(params)
      validate(params[:competition]) do |form|
        form.save
      end
    end
  end
end
```

```ruby
# app/concepts/competition/contract.rb
class Competition < ActiveRecord::Base
  module Contract
    class Create < Reform::Form
      model :competition

      property :name
      # [...] more properties

      validates :name, presence: true
    end
  end
end
```

Now, if I follow the Chapter 3 of the book, it wants me to refactor the `new` method (and it's view) and then it continues with the update part. But I don't want to do this until I have my form object complete with nested models. That is where it gets tricky.

### Nested forms: persisting belongs_to records

In the Trailblazer book, we need to jump to chapter 4 “Nested forms” that starts page 82.

First, I have a competition's `author`, which must be attributed to the `current_user`. It makes sense to me to use the `setup_model!` method (p. 85-86): the user is not part of the form, so dealing with the author relationship can be done at Operation level. This means, I need to pass the current user to my operation in the parameters:

```ruby
RSpec.describe Competition::Create do
  # [...]
  let!(:user) { FactoryGirl.create(:user) }

  it "creates a competition with author" do
    competition = Competition::Create
                  .call(competition: { name: "new competition" },
                        current_user: user)
                  .model

    expect(competition).to be_persisted
    expect(competition.author).to eq user
  end
end
```

```ruby
# app/controllers/competitions_controller.rb
def create
  run Competition::Create, 
      params: params.merge(current_user: current_user) do |op|
    return redirect_to op.model
  end

  render action: :new
end
```

```ruby
# app/concepts/competition/operation.rb
class Competition < ActiveRecord::Base
  class Create < Trailblazer::Operation
    include Model
    model Competition, :create

    contract Contract::Create

    def process(params)
      validate(params[:competition]) do |form|
        form.save
      end
    end

    private

      def setup_model!(params)
        model.author = params[:current_user]
      end
  end
end
```

A competition also belongs to 2 cities, `start_city` and `end_city` which params are passed in the request. That means, the `setup_model!` hook is not appropriate. Continuing reading the chapter 4, we learn about the `populate_if_empty` option for nested forms. We can pass a method to this option and this is going to be practical to setup our cities.

In somewherexpress app, cities are immutable objects. Whether the user creates or updates a City, it should look in the DB if there is an existing city with the same `locality` attribute, and if yes, set it as the corresponding city (`start_city` or `end_city`). If no, create a new City with the passed params.

Here is the updated test:

```ruby
RSpec.describe Competition::Create do
  # [...]
  let!(:user) { FactoryGirl.create(:user) }
  let!(:existing_city) do
    FactoryGirl.create(:city, locality: "Munich", name: "Munich, DE")
  end

  it "creates a published competition" do
    competition = Competition::Create
                  .call(competition: {
                          name: "new competition",
                          start_city: { name: "Yverdon, CH",
                                        locality: "Yverdon-Les-Bains",
                                        country_short: "CH" },
                          end_city: { name: "Munich, DE",
                                      locality: "Munich",
                                      country_short: "DE" }
                        },
                        current_user: user)
                  .model

    expect(competition).to be_persisted
    expect(competition.author).to eq user
    expect(competition.start_city.locality).to eq "Yverdon-Les-Bains"
    expect(competition.end_city.id).to eq existing_city.id
  end
end
```

This is how the contract evolves:

```ruby
# app/concepts/competition/contract.rb
class Competition < ActiveRecord::Base
  module Contract
    class Create < Reform::Form
      model :competition

      property :name
      # [...] more properties

      validates :name, presence: true

      property :start_city, populate_if_empty: :populate_city! do
        property :name
        property :locality
        property :country_short
        # [...] more properties

        validates :name, :locality, :country_short, presence: true
      end

      property :end_city, populate_if_empty: :populate_city! do
        property :name
        property :locality
        property :country_short
        # [...] more properties

        validates :name, :locality, :country_short, presence: true
      end

      private

        def populate_city!(options)
          # About this first return: it's a small hack. In my form view, I use
          # google places autcomplete. That means, when a name is entered,
          # a locality is set. But if the user erases the name, the locality
          # stays. The validation being ran only after populating, this first
          # return ensures that the form don't validate if the user erased the
          # name (but the locality stayed).
          return City.new unless options[:fragment].present? &&
                                 options[:fragment][:name].present?

          city = City.find_by(locality: options[:fragment][:locality])

          return city if city
          City.new(options[:fragment])
        end
    end
  end
end
```

And it works. It's as simple as this. I'm almost shocked. For you to understand, doing this with strong params and no deserializer nor form object was a huge pain: I wrote a dedicated service `Competitions::Update`, 100 lines of code to setup the correct `start_city`, `end_city` (and, as we will see later, each competition's tracks `start_city` and `end_city`). It was a complete hack, failing in edge cases.

It feels so nice and clean now. If the only gain was this part, it was worth using Reform!

As we can see, the 2 cities' properties are the same. We can extract this code to it's own form to make it cleaner:

```ruby
# app/concepts/city/form.rb
class City < ActiveRecord::Base
  class Form < Reform::Form
    property :name
    property :locality
    property :country_short
    # [...] more properties

    validates :name, :locality, :country_short, presence: true
  end
end
```

```ruby
# app/concepts/competition/contract.rb
class Competition < ActiveRecord::Base
  module Contract
    class Create < Reform::Form
      # [...]
      property :start_city, populate_if_empty: :populate_city!,
                            form: City::Form

      property :end_city, populate_if_empty: :populate_city!,
                          form: City::Form
      # [...]
    end
  end
end
```

Finally, I can introduce my second validation on competition which reads like this:

```ruby
validates :start_registration, :start_city, :end_city,
          :start_date, :end_date, presence: { if: :published? }
```

```ruby
RSpec.describe Competition::Create do
  # [...]
  let!(:user) { FactoryGirl.create(:user) }
  let!(:existing_city) do
    FactoryGirl.create(:city, locality: "Munich", name: "Munich, DE")
  end

  it "creates a published competition" do
    competition = Competition::Create
                  .call(competition: {
                          name: "new competition",
                          published: "1",
                          start_date: 2.weeks.from_now.to_s,
                          end_date: 3.weeks.from_now.to_s,
                          start_registration: Time.current,
                          start_city: { name: "Yverdon, CH",
                                        locality: "Yverdon-Les-Bains",
                                        country_short: "CH" },
                          end_city: { name: "Munich, DE",
                                      locality: "Munich",
                                      country_short: "DE" }
                        },
                        current_user: user)
                  .model

    expect(competition).to be_persisted
    expect(competition.author).to eq user
    expect(competition.start_city.locality).to eq "Yverdon-Les-Bains"
    expect(competition.end_city.id).to eq existing_city.id
  end
end
```

The trick is going to be the `if: :published?` part. Reform classes don't know about the Rails models magics, so we have to define `published?` explicitly. I found [this page](https://github.com/apotonick/reform/wiki/Conditional-validation-based-on-checkboxes) of the Reform documentation to be very helpful in that regard:

```ruby
# app/concepts/competition/contract.rb
class Competition < ActiveRecord::Base
  module Contract
    class Create < Reform::Form
      # [...]
      property :name
      property :start_date
      property :end_date
      property :start_registration
      property :published

      property :start_city, populate_if_empty: :populate_city!,
                            form: City::Form

      property :end_city, populate_if_empty: :populate_city!,
                          form: City::Form

      validates :name, presence: true
      validates :start_registration, :start_city, :end_city,
                :start_date, :end_date, presence: { if: :published? }
      # [...]

    private

      def published?
        published && published != "0"
      end

      # [...]
    end
  end
end
```

And that's it, we are good with competition level validations and belongs_to kind of nested models.

### Nested forms: persisting has_many records

A Competition has many tracks, and tracks are created and updated in the competition's form. A user can destroy a track from the form. This will make a delete request to `TracksController#destroy`, so we don't need to deal with removed tracks here. Tracks also have 2 nested cities, like Competition has. They are populated in the same way.

We now have to implement those nested tracks' properties in our contract. The rest of chapter 4 of the Trailblazer book won't help us much, it's mostly about rendering the form. We need to move to chapter 5 “Mastering Forms”, that starts page 128.

```ruby
RSpec.describe Competition::Create do
  # [...]
  let!(:user) { FactoryGirl.create(:user) }
  let!(:existing_city) do
    FactoryGirl.create(:city, locality: "Munich", name: "Munich, DE")
  end

  it "creates a published competition" do
    competition = Competition::Create
                  .call(competition: {
                          name: "new competition",
                          published: "1",
                          start_date: 2.weeks.from_now.to_s,
                          end_date: 3.weeks.from_now.to_s,
                          start_registration: Time.current,
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

    expect(competition).to be_persisted
    expect(competition.author).to eq user
    expect(competition.start_city.locality).to eq "Yverdon-Les-Bains"
    expect(competition.tracks.size).to eq 1
    expect(competition.tracks.first.end_city.id).to eq existing_city.id
    expect(competition.end_city.id).to eq existing_city.id
  end
end
```

For `has_many` kind of nested relationships, we can use a collection:

```ruby
# app/concepts/competition/contract.rb
class Competition < ActiveRecord::Base
  module Contract
    class Create < Reform::Form
      # [...]
      collection :tracks, populate_if_empty: :populate_track! do

        property :start_time

        property :start_city, populate_if_empty: :populate_city!,
                              form: City::Form

        property :end_city, populate_if_empty: :populate_city!,
                            form: City::Form

        validates :start_city, :end_city, :start_time, presence: true

        private

          def populate_city!(options)
            return City.new unless options[:fragment].present? &&
                                   options[:fragment][:name].present?

            city = City.find_by(locality: options[:fragment][:locality])

            return city if city
            City.new(options[:fragment])
          end
      end

      private

        def populate_track!(options)
          Track.new(start_time: options[:fragment][:start_time])
        end

      # [...]
    end
  end
end
```

And that is pretty much everything we want in terms of persisting incoming data.

### Put back authorization and callback

As I said earlier, I want to make the minimal refactor, that is, the form object. So let's put back the authorization method (I use pundit) and the callback (sending email) as they were originally.

For authorization, pundit's `authorize` method expects either an instance of competition, or the Competition class itself. In `CompetitionsController#create`, I don't have any instance of competition anymore. My policy does not depend on a competition record, only on the current user. That means I can pass the Competition class to authorize:

```ruby
# app/policies/competition_policy.rb
class CompetitionPolicy < ApplicationPolicy
  # [....]
  def create?
    user && (user.organizer || user.admin)
  end
end
```

```ruby
# app/controllers/competitions_controller.rb
def create
  authorize Competition
  # [...]
end
```

About the callback, I can simply move everything to the operation:

```ruby
class Competition < ActiveRecord::Base
  class Create < Trailblazer::Operation
    include Model
    model Competition, :create

    contract Contract::Create

    def process(params)
      validate(params[:competition]) do |form|
        form.save

        # This #published? will work: it's called on the AR model, which
        # understands this method call (unlike the form object)
        send_new_competition_emails if model.published?
      end
    end

    private

      def send_new_competition_emails
        User.want_email_for_new_competition.each do |user|
          UserMailer.as_user_new_competition(user.id, model.id).deliver_later
        end
      end

      # [...]
  end
```

The callback is still problematic: emails will be sent on each call of the Operation. But that's a problem for another time.

We still have a problem in `CompetitionsController#create`: if the form does not validate, we need to render `new`. But `new` requires a `@competition` object which we don't have anymore. We could hack it by passing the form as `@competition`.

In somewherexpress app case, it would require a few modifications of the layout and helpers, but not so many. I will show those changes when we refactor the rendering of the form. But it's just to say that it is doable without breaking the main structure of the view. This would be our final controller method:

```ruby
def create
  authorize Competition
  operation = run Competition::Create,
                  params: params.merge(current_user: current_user) do |op|
    return redirect_to op.model
  end

  @competition = operation.contract
  render action: :new
end
```

That's it! We have moved the persisting behavior of Competition to Trailblazer classes! And we did it without the need to refactor the entire application, or even the entire controller.

For this, I give you 👍👍 Trailblazer, you delivered on "you can refactor just some parts".

[Go to part 2]({% post_url 2016-08-22-journey-of-a-trailblazer-part-2 %})
