---
layout: post
title:  "PUB/SUB in Rails; using ActiveSupport::Notifications"
date:   2014-03-18 06:00:00
categories: [rails, pub/sub, design pattern, decoupling]
---

This article is about implementing simple PUB/SUB in Rails, using ActiveSupport::Notifications

TL;DR; You can write de-coupoled, compasable and modular application using simple pattern like PUB/SUB using ActiveSupport::Notifications.

- Why PUB/SUB?
- What is PUB/SUB?
- Implementing basic publisher & subscriber; lets get it on
	- Publisher
	- Subscriber
- Use cases where PUB/SUB can be helpful
	- Mail deliveries coupled with code
	- Messy callback chain introduced due to denormalization
	- Handling of events like user milestone reached, batchmate signed up
- Cocluding thoughts
- Request for gemification


# Why PUB/SUB?

Here at [Alma Connect](http://www.almaconnect.com/) we are heavy users of Ruby on Rails. RoR is a fast paced web development framework preferring convention over configuration. Rails features and initial project structure encourage fat model skinny controller approach. As the project evolves maintaining it becomes a real pain. Developers start using new patterns and libraries to cope up with these problems ([interactors](https://github.com/collectiveidea/interactor), [form objects](https://github.com/apotonick/reform), [decorators](https://github.com/drapergem/draper)). Different projects have different requirements and a lot of times people end up creating there own versions of solutions to problems faced by many. Today we will talk about use of pub/sub pattern to create a more de-coupeled system.

# What is PUB/SUB?
Pub/Sub (Publisher/Subscriber) is a very basic pattern with a lot variants available out there. Lets just get on the same page here and define the pub/sub we are talking about. Most of rails developers out there will be familiar with javascript and jquery. The pub/sub architecture we are talking about is very much similar to jQuery events. To capture the gist of the system:

  1. Publishers should be able to publish named events with payload
  2. Subscribers should be able to subscribe to events matching a name and should receive the payload object as well

# Implementing basic publisher & subscriber; lets get it on

Such a system is already bundled with rails: its [ActiveSupport::Notifications](http://api.rubyonrails.org/classes/ActiveSupport/Notifications.html)(ASN). ASN is used through out rails internally for instrumentation. The basic implementation of ASN is such a simple yet powerful API that we can build our pub/sub implementation on top of it.

## Publisher

{% highlight ruby linenos %}
# app/pub_sub/publisher.rb
module Publisher
  extend self
 
  def broadcast_event(event_name, payload={})
    if block_given?
      ActiveSupport::Notifications.instrument(event_name, payload) do
        yield
      end
    else
      ActiveSupport::Notifications.instrument(event_name, payload)
    end
  end
end
{% endhighlight %}

{% highlight ruby linenos %}
# publisher example usage
if user.save 
  Publisher.broadcast_event('user.created', user: user)
end

def create_user(params)
  user = User.new(params)
  
  Publisher.broadcast_event('user.created', user: user) do 
    User.save!
    # do some more important stuff here
  end
end
{% endhighlight %}

This is as simple as it gets. Publisher can broadcast an event accepting an event name and a payload hash. It can accept an optional block and will report the time taken to execute the block and also any error during execution of block(this is all provided by ASN used as base, interesting isn't it?).


## Subscriber

{% highlight ruby linenos %}
# app/pub_sub/subscriber.rb
module Subscriber
  def self.subscribe(event_name)
    if block_given?
      ActiveSupport::Notifications.subscribe(event_name) do |*args|
        event = ActiveSupport::Notifications::Event.new(*args)
        yield(event)
      end
    end
  end
end
{% endhighlight %}

{% highlight ruby linenos %}
# subscriber example usage
Subscriber.subscribe('user.created') do |event|
  error = "Error: #{event.payload[:exception].first}" if event.payload[:exception]
  puts "#{event.transaction_id} | #{event.name} | #{event.time} | #{event.duration} | #{event.payload[:user].id} | #{error}"
end
{% endhighlight %}

This is also pretty straight forward, to subscribe to an event just register with a name and the [ActiveSupport::Notifications::Event](http://api.rubyonrails.org/classes/ActiveSupport/Notifications/Event.html) object gets passed to the attached block as the parameter.

# Use cases where PUB/SUB can be helpful

These are some problems we face in our routine work, which can be solved by `pub_sub`:

  1. Mail deliveries coupled with code
  2. Messy callback chain introduced due to denormalization
  3. Handling of events like user milestone reached, batchmate signed up


## Mail deliveries coupled with code

**Problem:** There are always mail deliveries at work in any user facing project. Initially the place to send seems to be model callbacks. As the project evolves, you realize that a service object handling the whole process of user signup might be a better place for it. Lets explore usage of `pub_sub` in such scenario.

**Solution:** Technically our publisher and subscribers can still be used to handle such scenarios, but lets add some object oriented love and some ruby goodness to our implementation.

{% highlight ruby linenos %}
# app/pub_sub/publisher.rb
module Publisher
  extend ::ActiveSupport::Concern
  extend self

  included do
    class_attribute :pub_sub_namespace

    self.pub_sub_namespace = nil
  end


  def broadcast_event(event_name, payload={})
    if block_given?
      self.class.broadcast_event(event_name, payload={}) do
        yield
      end
    else
      self.class.broadcast_event(event_name, payload={})
    end
  end


  module ClassMethods
    def broadcast_event(event_name, payload={})
      event_name = [pub_sub_namespace, event_name].compact.join('.')
      if block_given?
        ActiveSupport::Notifications.instrument(event_name, payload) do
          yield
        end
      else
        ActiveSupport::Notifications.instrument(event_name, payload)
      end
    end
  end

end
{% endhighlight %}

The publisher undergoes some changes, to be included in class and have a class level namespace. Lets say we are talking about welcome mails here: lets just create a publisher for registrations and start publishing

{% highlight ruby linenos %}
# app/pub_sub/publishers/registration.rb
module Publishers
  class Registration
    include Publisher

    self.pub_sub_namespace = 'registration'
  end
end

Publishers::Registration.broadcast_event('user_signed_up', user: user)
{% endhighlight %}


Why not make our subscribers more class friendly to? Our old subscriber remains unchanged, but we have a new base subscriber class:

{% highlight ruby linenos %}
# app/pub_sub/subscribers/base.rb
module Subscribers
  class Base
    class_attribute :subscriptions_enabled
    attr_reader :namespace

    def initialize(namespace)
      @namespace = namespace
    end


    def self.attach_to(namespace)
      log_subscriber = new(namespace)
      log_subscriber.public_methods(false).each do |event|
        ActiveSupport::Notifications.subscribe("#{namespace}.#{event}", log_subscriber)
      end
    end

    def call(message, *args)
      method  = message.gsub("#{namespace}.", '')
      handler = self.class.new(namespace)
      handler.send(method, ActiveSupport::Notifications::Event.new(message, *args))
    end
  end
end
{% endhighlight %}

What we did here was create a base class to be extended. A subscriber can attach itself to a namespace and define methods to handle individual events. Method names should match with event name in the namespace.

{% highlight ruby linenos %}
# app/pub_sub/subscribers/registration_mailer.rb
module Subscribers
  class RegistrationMailer < ::Subscribers::Base
    def user_signed_up(event)
      # lets delay the delivery using delayed_job
      RegistrationMailer.delay(priority: 1).welcome_email(event.payload[:user])
    end
  end
end

# config/initializers/subscribers.rb
Subscribers::RegistrationMailer.attach_to('registration')
{% endhighlight %}

Nothing is really happening until the call to `attach_to`, someone is broadcasting event that a user has signed up, but nobody is interested in the event, so it goes unnoticed. If a subscriber subscribes to the event, interesting things can be done, like sending out that welcome email :)

**Conclusion:** With little modifications we have built our system to handle different modules and features. Adding a layer on top of already useful `ActiveSupport::Notification` already feels good enough.


## Messy callback chain introduced due to denormalization

**Problem:** We use mongodb with mongoid heavily and denormalize data to cut down on queries. We have built an internal system to handle denormalization with decent fallbacks and keeping the sync in realtime. It has been working out very nicely, but it results in very coupled code. We define denormalization macros in the models. This makes all our models aware of what data they are denormalizing from where and also what data they need to denormalize to where. The code looks something like:

{% highlight ruby linenos %}
# app/models/user.rb
class User
  include AlmaConnect::Denormalization
  field :name, type: String
  has_many :posts

  # this sets up a callback for when name changes,
  # it propagates changes to posts where user_id matches the user
  denormalize_to :posts, fields: [:name]
end

# app/models/post.rb
class Post
  include AlmaConnect::Denormalization
  belongs_to :user

  # this sets up a callback to update user_name when user_id changes
  # also fetch user name from user if it is not already set
  denormalize_from :user, fields: [:name]
end
{% endhighlight %}

This looks very good, but what if we need to denormalize user name to comment as well.

{% highlight ruby linenos %}
# app/models/comment.rb
class Comment
  include AlmaConnect::Denormalization
  embedded_in :post
  belongs_to :user, inverse_of: nil
  denormalize_from :user, fields: [:name]
end

# app/models/post.rb
class Post
  include AlmaConnect::Denormalization
  belongs_to :user
  embeds_many :commments
  denormalize_from :user, fields: [:name]
end

# app/models/user.rb
class User
  include AlmaConnect::Denormalization
  field :name, type: String
  has_many :posts
  denormalize_to :posts, fields: [:name]

  # we don't define the relation here, still define the denormalization macro
  denormalize_to 'post.comments', fields: [:name]
end
{% endhighlight %}

You see the problem, right?

**Solution:** Instead of sprinkling denormalization macros all around, we can leave the from macros as they are. From macros are good, because of all the fallback logic, they also work on pull content from a model rather push changes to many. We can replace to macros, with publishing change events. We would simply publish a message whenever a user name changes. We can capture it in a `before_save` callback and publish in a `after_save`. By publishing in `after_save` we ensure that user is persisted at time of publishing. We can change the publisher to hook directly into model callback chain.

{% highlight ruby linenos %}
# app/pub_sub/publishers/base.rb
module Publishers
  module Base
    extend ActiveSupport::Concern

    def pub_sub_notifications
      @pub_sub_notifications ||= ::Publishers::PubSubNotifications.new(self)
    end

    module ClassMethods
      def attach_publisher(namespace, publisher_class)
        after_initialize do |model|
          model.pub_sub_notifications.attach_publisher(namespace, publisher_class)
        end

        before_save do |model|
          model.pub_sub_notifications.prepare_created(namespace)
          model.pub_sub_notifications.prepare_notifications(namespace)
        end

        after_save do |model|
          model.pub_sub_notifications.publish_notifications(namespace)
        end

        after_destroy do |model|
          model.pub_sub_notifications.prepare_destroyed(namespace)
          model.pub_sub_notifications.publish_notifications(namespace)
        end
      end
    end

  end
end

# app/pub_sub/publishers/notifications_queue.rb
module Publishers
  class NotificationsQueue
    attr_reader :publisher, :notifications

    def initialize(publisher_name)
      @publisher = publisher_name.to_s.constantize.new
      @notifications = []
    end

    def add_notification(event_name, payload={})
      @notifications << {event_name: event_name, payload: payload}
    end

    def reset_notifications
      @notifications = []
    end
  end
end

# app/pub_sub/publishers/pub_sub_notifications.rb
module Publishers
  class PubSubNotifications
    include ::Publisher

    attr_reader :publishers_info, :model

    def initialize(model)
      @publishers_info = {}
      @model               = model
    end

    def attach_publisher(namespace, publisher_name)
      publishers_info[namespace] ||= Publishers::NotificationsQueue.new(publisher_name)
      true
    end

    def reset_notifications(namespace)
      publishers_info[namespace].reset_notifications
    end

    def add_notification(namespace, event_name, payload={})
      publishers_info[namespace].add_notification(event_name, payload)
    end

    def prepare_created(namespace)
      add_notification(namespace, 'created') if model.new_record?
    end

    def prepare_destroyed(namespace)
      add_notification(namespace, 'destroyed') if model.destroyed?
    end

    def prepare_notifications(namespace)
      publishers_info[namespace].publisher.prepare_notifications(namespace, model)

      return true
    end

    def publish_notifications(namespace)
      publishers_info[namespace].notifications.each do |notification|
        broadcast_event(
            [namespace, notification[:event_name]].compact.join('.'),
            notification[:payload].merge(model: model)
        )
      end

      return true
    end

  end
end
{% endhighlight %}

We created a module `Publishers::Base` which will be included in an `ActiveModel`. Model can now `attach_publisher(namespace, publisher_klass)` and immediately start broadcasting events `created` and `destroyed` in that namespace. The publisher in the call needs to define a method `prepare_notifications(namespace, model)`. This method will be called when ever `before_save` is triggered on the model instance. Publisher can access the `pub_sub_notifications` item which is a registry of all the publishers attached to the model and add any notifications which would be broad casted as events in the `after_save`. Each `attach_publisher` call creates a new object of notifications_queue, which creates a new instance of publisher on every callback cycle and maintains an array of notifications to be published for this publisher. We initially implemented it so that publishers were registered in an initializer like subscribers below. It created many issues with development file reloads and constant restarts of server and console. Publishers are the part where all the actions happens. Maintaining list of publishers, their namespace, notifications to be published are managed here. Subscribers are simple listeners which work on per event basis. We need our notification system to be as thread safe as usage of model instances themselves, hence the complexity.

{% highlight ruby linenos %}
# app/pub_sub/publishers/user.rb
module Publishers
  module User
    def prepare_notifications(namespace, user)
      if user.name_changed?
        user.pub_sub_notifications.add_notification(namespace, 'name_changed', changes: user.name_changes)
      end
    end
  end
end

# app/models/user.rb
class User
  include Publishers::Base

  attach_publisher('user', Publishers::User)
end

class Subscribers::User::Post < Subscribers::Base
  def name_changed(event)
    # TODO: delay this update :)
    Post.where(user_id: event.payload[:model].id).update_all(user_name: event.payload[:changes].last)
  end
end

class Subscribers::User::Comment < Subscribers::Base
  def name_changed(event)
    # TODO: defenitely delay these iterations ;)
    def name_changed(event)
      iteration_limit = 100 # or fetch max comment count
      posts = Post.all.where(:comments.matches => {user_id: event.payload[:model].id, :user_name.ne => event.payload[:changes].last})
      while iteration_limit > 0 && posts.count > 0
        # TODO: Now jokes apart, we seriously need to at least upgrade to mongoid 3
        # before we plan to move on to update rails 4 and mongoid 4
        Post.collection.update(posts.selector, {'$set' => {'comments.$.user_name' => event.payload[:changes].last} }, multi: true, safe: true)
      end
    end
  end
end

# config/initializers/subscribers.rb
Subscribers::User::Post.attach_to('user')
Subscribers::User::Comment.attach_to('user')
{% endhighlight %}

This is a lot more code, but this system seems robust:

  - Easy to add new denormalization points
  - Complete control of how to handle the change
  - A lot more modular system
  - Easily take out some subscriber or publisher without affecting the system
  - Individual sub-components can be enabled, disabled and changed independently

**Conclusion:** Adding some more ruby goodness to the mix, we have built a production grade iron to straighten out the callback spaghetti we have been cooking recently.

## Handling of events like user milestone reached, batchmate signed up

**Problem:** There are other kind of things we need to do here at alma connect. We have mail triggers to things like user count milestone reached, batchmates signed up or updated important profile info. How do we implement such things?

**Solution:** Luckily for us we already had a steady `pub_sub` implementation by the time we needed to do this. We created a very modular and layered architecture. and wired it together using the `pub_sub`. We could already listen to `user.created` event from our user publisher. We can create milestone and batchmate signed up event from this event and handle them appropriately.

{% highlight ruby linenos %}
# app/services/registration/user_notifications_service.rb
class Registration::UserNotificationsService
  def self.deliver_welcome_email(user)
    RegistrationMailer.welcome(user).deliver
  end

  def self.prepare_batchmate_notification(batchmate)
    User.batchmates_of(batchmate).each do |user|
      Publishers::Registration.broadcast_event('deliver_batchmate_signed_up', batchmate: batchmate, user: user)
    end
  end

  def self.deliver_batchmate_signed_up_email(batchmate, user)
    RegistrationMailer.batchmate_signed_up(user, batchmate).deliver
  end
end

# app/jobs/welcome_email_job.rb
class WelcomeEmailJob
  def initialize(user)
    @user = user
  end

  def perform
    Registration::UserNotificationsService.deliver_welcome_email(user)
  end

  def self.deliver(user)
    Delayed::Job.enqueue(new(user))
  end
end

# app/jobs/batchmate_activity_job.rb
class BatchmateActivityJob
  def initialize(batchmate, user=nil)
    @batchmate = @batchmate
    @user = user
  end

  def perform
    if bundled?
      Registration::UserNotificationsService.prepare_batchmate_notification(@user, @batchmate)
    else
      Registration::UserNotificationsService.deliver_batchmate_signed_up_email(@user, @batchmate)
    end
  end

  def bundled?
    @user.nil?
  end

  def self.prepare(batchmate)
    Delayed::Job.enqueue(new(batchmate))
  end

  def self.deliver(batchmate, user)
    Delayed::Job.enqueue(new(batchmate, user))
  end
end
{% endhighlight %}

Till now we have created everything we need to send out welcome email and milestone notification. We can see that the jobs expose three methods and they correspond to three methods in our notification service. Notification service is the place where all the real action is going on. Actual mail delivery and applying the business logic to identify the batchmates and triggering the delivery all lives in there. These all are activities which can be delayed, so we have created two jobs, each handling delaying of welcome emails and milesonte notification. We can now write the subscribers and and wire everything up.

{% highlight ruby linenos %}
# app/pub_sub/subscribers/registration/user.rb
class Subscribers::Registration::User < Subscribers::Base
  def created(event)
    WelcomeEmailJob.deliver(user)
    BatchmateActivityJob.prepare(user)
  end
end

module Subscribers
  class RegistrationMailer < ::Subscribers::Base
    def deliver_batchmate_signed_up(event)
      BatchmateActivityJob.deliver(event.payload[:batchmate], event.payload[:user])
    end
  end
end

# config/initializers/subscribers.rb
Subscribers::Registration::User.attach_to('user')
Subscribers::RegistrationMailer.attach_to('registration')
{% endhighlight %}

**Conclusion:** This wires everything up. Cool isn't it. We have built a modular and composable system. We gain a lot of flexibility and kick out a lot of coupled code. I like where all this is going.



# Cocluding thoughts

A simple pattern like pub/sub can be used to write highly decoupled modules which can be upgraded/added/taken out without affecting the rest of the system too much. It helps us write more modular system, which is highly composable. Additional layers can be added in later to accommodate further complexity.

In this article we have explored using basic pattern of pub/sub in our routine work and how it helps us write modular, decoupled and composable application. I have few things in mind: more on composability of pub/sub, going back to exploring instrumentation using pub/sub. We are also working on rethinking notification settings and strategy at AlmaConnect. Another topic of talk can be building loosely coupled and testable applications using pub/sub. There is always something going on at AlmaConnect. If you liked today's article, keep looking for the next one.

# Request for gemification

I am no expert here and might be missing some important issues. I welcome constructive criticism. Also if someone finds this interesting and wants to wrap this in a gem, please get in touch with me at `rubish[dot]gupta[at]gmail[dot]com`.