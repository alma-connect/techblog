---
layout: post
title:  "Layouts and rendering in rails"
date:   2015-02-01 06:00:00
categories: [rails, layouts]
---

I came across this [question](http://stackoverflow.com/questions/28272806/can-view-not-extend-application-application-html-erb-in-ruby-on-rails) on [stackoverflow](http://stackoverflow.com). There is an excellent guide out there about [Layouts and Rendering](http://guides.rubyonrails.org/layouts_and_rendering.html) which everyone should read. I would like to tell few tricks that can be used to support common scenarios.

- [Conditional content in layout](#conditional-content)
- [Multiple layouts](#multiple-layouts)
- [Common head in multiple layouts](#common-head)
- [Page specific assets](#page-specific-assets)

# <a name="contional-content">Conditional Content in Layout</a>

Lets start with a simple `application.html`:

{% highlight ruby linenos %}
# app/views/layouts/application.html.erb
<!DOCTYPE html>
<html>
  <head>
    <!-- head content -->
  </head>
  <body>
    <%= render 'layouts/header' %>
    <%= yield %>
  </body>
</html>
{% endhighlight %}

If header is shown only for some pages, rendering of header can be made conditional. It just requires a few modifications:

{% highlight ruby linenos %}
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  attr_accessor :headless
  helper_method :headless

  def show
    self.headless = true
  end
end
{% endhighlight %}


{% highlight ruby linenos %}
# app/views/layouts/application.html.erb
.....
  <%= render('layouts/header') unless headless %>
.....
{% endhighlight %}

We defined an attribute `headless` and also declared the getter to be helpor method, to facilitate usage in views. `headless` is set to `true` if we do not want to show header on a page.


# <a name="multiple-layouts">Multiple Layputs</a>

But that is not enough, registration and authenticated pages have a completely different layout. We cannot just keep adding conditionals. Lets create different layouts for out guests and users. We will keep application layout for users and create a new layout guest for guests.

{% highlight ruby linenos %}
# app/views/layouts/application.html.erb
<!DOCTYPE html>
<html>
  <head>
    <!-- head content -->
  </head>
  <body>
    <%= render('layouts/header') unless headless %>
    <div class="page">
      <%= render 'layouts/sidebar' %>

      <div class="page-content">
        <%= yield %>
      </div>
    </div>
  </body>
</html>
{% endhighlight %}

{% highlight ruby linenos %}
# app/views/layouts/guest.html.erb
<!DOCTYPE html>
<html>
  <head>
    <!-- head content -->
  </head>
  <body>
    <%= render 'layouts/login_bar' %>
    <%= yield %>
  </body>
</html>
{% endhighlight %}

{% highlight ruby linenos %}
# app/controllers/guest_controller.rb
class GuestController < ApplicationController
  # change layout for all controller actions 
  layout 'guest'

  def signin
  end

  def signup
    # change layout on per action basis
    render layout: 'guest'
  end
end
{% endhighlight %}

Layout can be configured on a controller level or per action level. If set on a controller level, all the actions of controller will use that layout. 


# <a name="common-head">Common head in multiple layouts</a>

Head content in both the layouts is common, why don't we extract it out in a partial. Extracting head content in a partial is trivial, but what if we want to extract out doctype and html tag as well? Worry not, partial layouts to the rescue.

{% highlight ruby linenos %}
# app/views/layouts/_head.html.erb
<!DOCTYPE html>
<html>
  <head>
    <!-- head content -->
  </head>
  <body>
    <%= yield %>

    <%= render 'layouts/footer' %>
  </body>
</html>
{% endhighlight %}

{% highlight ruby linenos %}
# app/views/layouts/application.html.erb
<%= render :layout => 'layouts/head' do %>

  <%= render('layouts/header') unless headless %>
  <div class="page">
    <%= render 'layouts/sidebar' %>

    <div class="page-content">
      <%= yield %>
    </div>
  </div>

<% end # render partial layout 'head' %>
{% endhighlight %}

{% highlight ruby linenos %}
# app/views/layouts/guest.html.erb
<%= render :layout => 'layouts/head' do %>

  <%= render 'layouts/login_bar' %>
  <%= yield %>        

<% end # render partial layout 'head' %>
{% endhighlight %}

Now we don't have to worry about making changes in multiple layouts when we change head.

# <a name="page-specific-assets">Page specific assets</a>

Now that we have one layout each for our guests and users, we can also saperate out our assets for the layouts. We can have template specific assets as well. `content_for` is apt for this.

{% highlight ruby linenos %}
# app/views/layouts/_head.html.erb
<!DOCTYPE html>
<html>
  <head>
    <!-- head content -->

    <%= yield :layout_assets %>
    <%= yield :template_assets %>
  </head>
  <body>
    <%= yield %>

    <%= render 'layouts/footer' %>
  </body>
</html>
{% endhighlight %}


{% highlight ruby linenos %}
# app/views/layouts/guest.html.erb
<%= render :layout => 'layouts/head' do %>
  <% content_for :layout_assets do %>
    <!-- layout specific assets -->
  <% end %>

  <%= render 'layouts/login_bar' %>
  <%= yield %>        

<% end # render partial layout 'head' %>
{% endhighlight %}

{% highlight ruby linenos %}
# app/views/guest/login.html.erb
<% content_for :template_assets do %>
  <!-- template specific assets -->
<% end %> 

<!-- template content -->
{% endhighlight %}

I hope this was useful imformation for some. If you have any tricks, please share in comments.
