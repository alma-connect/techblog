---
layout: post
title:  "Implementing PUB/SUB in Rails; using ActiveSupport::Notifications"
date:   2014-03-18 06:00:00
categories: [rails, pub/sub, design pattern, decoupling]
---

This artcile is about implmenting a simple publisher/subscriber model in rails using [ActiveSupport::Notifications](http://api.rubyonrails.org/classes/ActiveSupport/Notifications.html) (ASN). Exploring problem areas where PUBSUB can improve modularity and reduce coupling. We would also upgrade our basic implementation to cater to common use cases.

- [A little about PUB/SUB?](#about-pub-sub)
- [Implementing basic publisher & subscriber; lets get it on](#basic-implementation)
	- [Publisher](#basic-publisher)
	- [Subscriber](#basic-subscriber)
- [Use cases where PUB/SUB can be helpful](#use-cases)
	- [Mail deliveries coupled with code](#problem-mail-delivery-coupling)
	- [Messy callback chain introduced due to denormalization](#problem-callbacks-in-denormalized-models)
	- [Handling of events like user milestone reached, batchmate signed up](#problem-handling-special-events)
- [Cocluding thoughts](#concluding-thoughts)


# <a name="about-pub-sub">A little about PUB/SUB?</a>

According to [wikipedia](http://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) is: 

> In software architecture, publish–subscribe is a messaging pattern where senders of messages, called publishers, do not program the messages to be sent directly to specific receivers, called subscribers. Instead, published messages are characterized into classes, without knowledge of what, if any, subscribers there may be. Similarly, subscribers express interest in one or more classes, and only receive messages that are of interest, without knowledge of what, if any, publishers there are.

Rails is a good framework for proof-of-concept and small applications, it promotes conventions over configuration. As the application grows it becomes difficult to maintain. This is primarily because rails promotes coupeled architecture with fat model, skinny controller approach. Developers need to adopt to new patterns and create new conventions to keep projects moving on pace and in shape. PUBSUB can help us resolve some of the problems we face often.

# <a name="basic-implementation">Implementing basic publisher & subscriber; lets get it on</a>

Baisc requirements of our PUBSUB are:

  1. Publishers should be able to publish named events with payload
  2. Subscribers should be able to subscribe to events matching a name and should receive the payload object with every event

Such a system is already bundled with rails: its [ActiveSupport::Notifications](http://api.rubyonrails.org/classes/ActiveSupport/Notifications.html)(ASN). ASN is used through out rails internally for instrumentation. The basic implementation of ASN is such a simple yet powerful API that we can start our PUBSUB implementation on top of it.

## <a name="basic-publisher">Publisher</a>

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

This is as simple as it gets. Publisher can broadcast an event accepting an event name and a payload hash. It can accept an optional block and will report the time taken to execute the block and also any error during execution of block (this is all provided by ASN used as base, interesting!!).


## <a name="basic-subscriber">Subscriber</a>

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

# <a name="use-cases">Use cases where PUB/SUB can be helpful</a>

These are some problems we face in our routine work, which can be solved by PUBSUB:

  - [Mail deliveries coupled with code](#problem-mail-delivery-coupling)
  - [Messy callback chain introduced due to denormalization](#problem-callbacks-in-denormalized-models)
  - [Handling of events like user milestone reached, batchmate signed up](#problem-handling-special-events)

## <a name="problem-mail-delivery-coupling">Mail deliveries coupled with code</a>

**Problem:** There are always some kind of notifications at work in any user centeric application. Generally it starts with `after_save` model callback and slowly moves out of there to a services layer. Lets explore, how can we handle welcome emails to newly registered user using PUBSUB.

**Solution:** Since we are adding this pattern to our application, lets bake in the modularity in core usage.

{% highlight ruby linenos %}
# app/pub_sub/publisher.rb
module Publisher
  extend ::ActiveSupport::Concern
  extend self

  included do
    # add support for namespace, one class - one namespace
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

The publisher under went some changes. It can be included in a class and after setting a namespace, you are all set to publish events in that namespace. We are not sure yet who might be interested, but lets just start broascasting event `user_signed_up` in `registration` namespace.

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

That was easy, publishers look good. Lets add some object oriented love and some ruby magic to subscribers too.

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

We creeate a base class which subscribers can extend. Subscriber can then attach its method to events in a namespace. This seems to be the building block of highly modular application.

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

We have been broadcasting that event about user sign up, but nothing was really happening as a result. We created a subscriber to do something useful when the event is captured; send out that welcome email we have been talking about.

There is a gotcha though, building modular, losely coupeled system with message publishing has its own cons. Biggest being, the system is too loosely coupeled. If something stops working nothing goes wrong and it might take too long to identify it. However I believe that failures can happen anywhere and a system which fails with grace is more robust that one which fails completely. Its another layer which gets introduced while building scalable systems. **Base Line:** for me pros out weigh the cons

**Conclusion:** With little modifications, our PUBSUB now supports namespaces (read features, modules, sub-systems, opportunities!!). Adding a layer on top of already useful `ActiveSupport::Notification` already feels good enough.


## <a name="problem-callbacks-in-denormalized-models">Messy callback chain introduced due to denormalization</a>

**Problem:** We use mongodb via mongoid and denormalize data to cut down on queries. Intially it was some callbacks and method ooverrides, which grew out to be modules and then a simple re-usable denormalization plugin. It fall backs to querying data from related model and also cache it for future use. This works like a cache fetch or calculate & store strategy with remote document as origin and current document as cache store. The code looks something like:

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
  # also fetch and set, user name from user if it is not already set, when accessed
  denormalize_from :user, fields: [:name]
end
{% endhighlight %}

It has been working out very nicely, but it results in very coupled code. We define denormalization macros in the models. This makes all our models aware of what data they are denormalizing, where from and also what data they need to denormalize, where to. What if we need to denormalize user name to comment as well.

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

**Solution:** Instead of sprinkling denormalization macros all around, we can leave the from macros as is, they implement the falback logic which comes in very handy. We can replace to macros, with publishing change events. We would simply publish a message whenever a user name changes. We can capture it in a `before_save` callback and publish in a `after_save`. By publishing in `after_save` we ensure that user is persisted at time of publishing. We can change the publisher to hook directly into model callback chain.

{% highlight ruby linenos %}
# app/pub_sub/publishers/base.rb
# hook into model callback chain when a publisher is attached to a model 
# capture changes before_save and publish in after_save
# start publishing created and destroyed events
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
# capture attachment of a publisher to a model in a namespace
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
# handle different attachments of publishers to a model
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

We created a module, which can be included in any active model complaint model, supporting callbacks. Model can attash publishers to itself against a namespace. Attached publisher immediately starts broadcasting created and destroyed for the model. To broadcast additional events, publisher should override the `prepare_notifications` method and make calls to `model.pub_sub_notifications.add_notification`.

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
    Post.where(user_id: event.payload[:model].id).update_all(user_name: event.payload[:changes].last)
  end
end

class Subscribers::User::Comment < Subscribers::Base
  def name_changed(event)
    def name_changed(event)
      iteration_limit = 100 # or fetch max comment count
      posts = Post.all.where(:comments.matches => {user_id: event.payload[:model].id, :user_name.ne => event.payload[:changes].last})
      while iteration_limit > 0 && posts.count > 0
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

## <a name="problem-handling-special-events">Handling of events like user milestone reached, batchmate signed up</a>

**Problem:** A user signs up, this can potentially trigger a user milestone and would most probably trigger batchamted signed up notification. Where do we keep this code? In the service concerned with creation of user? In model callback? Nothing seems to fit the bill.

**Solution:** Lucky for us, we already have our PUBSUB. We can listen to `user.created` event from our user publisher and trigger approriate behaviour in a subscriber. Sending welcome mail and batchmate activity is not core to the application and should not effect core if smtp is not working at the moment.

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

Till now we have created everything we need to send out welcome email and milestone notification. We can see that the jobs expose three methods and they correspond to three methods in our notification service. Notification service is the place where all the real action is going on, all the business logic lives there. Notification service methods should be delayed, so we have created two jobs, each handling delaying of welcome emails and milesonte notification. We can now write the subscribers and and wire everything up.

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

**Conclusion:** This wires everything up. Cool isn't it. We have built a modular and composable system. We gain a lot of flexibility and get a change to phase out some bad code.



# <a name="concluding-thoughts">Cocluding thoughts</a>

A simple pattern like pub/sub can help us write highly decoupled modules. These loosely coupeled modules can be composed together in variety of ways to create a flexible, robust and scalable application. The modules can be replaced and upgraded without affecting the system as whole.

If you liked today's article, keep looking for the next one. If you like the implmentation and would love to help in gemifying it, please get in touch at `rubish[dot]gupta[at]gmail[dot]com` or `tech[dot]team[at]gmail[dot]com`
