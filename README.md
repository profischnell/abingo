A/Bingo Version 1.0.3
=====================

Rails A/B testing.  One minute to install.  One line to set up a new A/B test.
One line to track conversion.

For usage notes, see: [http://www.bingocardcreator.com/abingo](http://www.bingocardcreator.com/abingo)

Forked from the original distribution for easier integration into rails at [Toptranslation GmbH Übersetzungsagentur](https://www.toptranslation.com/).
[Profi-Schnell Übersetzungsagentur](http://www.profischnell.com).
Installation instructions are below usage examples.

Key default features:

* Conversions only tracked once per individual.
* Conversions only tracked if individual saw test.
* Same individual ALWAYS sees same alternative for same test.
* Syntax sugar. Specify alternatives as a range, array, hash of alternatives with weights, or just let it default to true or false.
* A simple z-test of statistical significance, with output so clear anyone in your organization can understand it.

Example: View
-------------

    <% ab_test("login_button", ["/images/button1.jpg", "/images/button2.jpg"]) do |button_file| %>
      <%= img_tag(button_file, :alt => "Login!") %>
    <% end %>

Example: Controller
-------------------

    def register_new_user
      #See what level of free points maximizes users' decision to buy replacement points.
      @starter_points = ab_test("new_user_free_points", [100, 200, 300])
    end

Example: Controller
-------------------

    def registration
      if (ab_test("send_welcome_email"), :conversion => "purchase")
        #send the email, track to see if it later increases conversion to full version
      end
    end

Example: Conversion tracking (in a controller!)
-----------------------------------------------

    def buy_new_points
      #some business logic
      bingo!("buy_new_points")  #Either a conversion named with :conversion or a test name.
    end

Example: Conversion tracking (in a view)
----------------------------------------

    Thanks for signing up, dude! <% bingo!("signup_page_redesign") >

Example: Statistical Significance Testing
-----------------------------------------

    Abingo::Experiment.last.describe_result_in_words
    => "The best alternative you have is: [0], which had 130 conversions from 5000 participants (2.60%).
        The other alternative was [1], which had 1800 conversions from 100000 participants (1.80%).
        This difference is 99.9% likely to be statistically significant, which means you can be extremely
        confident that it is the result of your alternatives actually mattering, rather than being due to
        random chance.  However, this doesn't say anything about how much the first alternative is really
        likely to be better by."

Installation
============

Configure the Gem
-----------------

    gem 'abingo', :git => "git://github.com/moeffju/abingo.git"
    bundle install 

Generate the database tables 
----------------------------
Creates tables "experiments" and "alternatives". If you use these names already you will need to do some hacking)

    rails g abingo_migration
    rake db:migrate

Configure a Cache
-----------------
A/Bingo makes HEAVY use of the cache to reduce load on thedatabase and share potentially long-lived "temporary" data, such as what alternative a given visitor should be shown for a particular test.  

A/Bingo defaults to using the same cache store as Rails.These instructions are on how to use the memcache-addon in heroku.

    heroku addons:add memcache:5mb

    #Gemfile
    group :production do
      gem "memcache-client"
      gem 'memcached-northscale', :require => 'memcached'
    end

    #config/environments/production.rb
    # Use a different cache store in production
    # config.cache_store = :mem_cache_store
    config.cache_store = :mem_cache_store, Memcached::Rails.new

Tell A/Bingo a user's identity
------------------------------
So abingo knows who a users is if they come back to a test.  (The same identity will always see the same alternative for the same test.)  How you do this is up to you -- I suggest integrating with your login/account infrastructure.  The simplest thing that can possibly work is:

    #Somewhere in application.rb
    before_filter :set_abingo_identity

    def set_abingo_identity
      #treat all bots as one user to prevent skewing results
      if request.user_agent =~ /\b(Baidu|Gigabot|Googlebot|libwww-perl|lwp-trivial|msnbot|SiteUptime|Slurp|WordPress|ZIBB|ZyBorg)\b/i  
         Abingo.identity = "robot"
       elsif current_user
         Abingo.identity = current_user.id
       else
         session[:abingo_identity] ||= rand(10 ** 10) 
         Abingo.identity = session[:abingo_identity]
       end
    end

Create the Dashboard
--------------------
You need to create a controller which includes the methods from the Abingo module, as well as generate the views. You can customise the view if you wish.
Don't forget to authenticate access to the controller.

    rails g controller admin/abingo_dashboard
    
    #app/controllers/admin/abingo_dashboard_controller.rb
    class Admin::AbingoDashboardController < ApplicationController
      include Abingo::Controller::Dashboard
    end
    
    #routes.rb
    namespace :admin do
      get "ab_dashboard"   => "abingo_dashboard#index"
      post "ab_end_experiment/:id"   => "abingo_dashboard#end_experiment"
    end
    
    rails g abingo_views
    
Run your first test!
====================

Copyright
=========

Copyright (c) 2009-2010 Patrick McKenzie, released under the MIT license
