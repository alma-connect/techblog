---
layout: post
title:  "Alma Connect TODO projects; call for interns"
date:   2014-04-05 06:00:00
categories: [logstash, elasticsearchm, kibana, mongoid, mongodb, rails, intern, todo]
---

I would admit that [Alma Connect](http://www.almaconnect.com) codebase is not in its best shape. The culprit is evolving project and software erosion. You can read the brilliant post by heroku on [erosion resistance](https://blog.heroku.com/archives/2011/6/28/the_new_heroku_4_erosion_resistance_explicit_contracts). Evolving project is very subjective issue. In a startup you want to build fast, release fast, tweak fast. All this urgency created due to fast is too much. Often times you intentionally let bad code slip into the codebase to save time.

Today we will talk about the things which we want TODO at alma connect, but can not due to the fast pace of the world. We are looking for interns for these projects and we promise to provide a very good learning experience.

- [Setup analytics engine](#setup-analytics-engine)
- [Grape for composable api end points in rails](#grape-api-rails)
- [Decoupling frontend from backend](#decouple-frontend)
- [Many more projects](#many-more-projects)
    - [Upgrade Mongoid and dependencies](#upgrade-mongoid)
    - [Testing framework](#testing-framework)
    - [Component structure on backend](#backend-modules)
- [Internship call out](#intern-call)


# <a name="setup-analytics-engine">Setup analytics engine</a>

We use googly analytics (GA) for our analytics needs, but GA doesn't provide us the raw data. Without raw data there are limitations on things that can be discovered. Recently we analyzed the referral request data from alma connect database, using simple tool like pivot table in excel and were able to gain pretty good insights about the process and areas for improvement.

We are looking to build a analytics engine where we can dump any information and analyze easily whenever needed. We want to start small, but want a easily scalable system. We are looking at [ELK stack](http://www.elasticsearch.org/overview/elkdownloads/). Logstash will capture events from log files, app servers, background worker, frontend application. Captured events will be forwarded to s3 for archiving and to elasticsearch for real time analytics. Kibana will be used to visualize data on ad hoc bases and start playing with new data straight away. We would also create an elasticsearch site plugin to support any custom queries or reports.

# <a name="grape-api-rails">Grape for composable api end points in rails</a>

Authentication and authorization are two very different things. A lot of things can be controlled by authorization, but not everything. When there are different types of users with different needs they should have dedicated views for them. A composable data api for these views must be inplace before building the view on top of it.

We are thinking of implementing end points pertaining to each feature or module using the [grape](https://github.com/intridea/grape) gem family. We would be able to build composable robust APIs which can be mounted and multiple endpoints like: user, admin, recruiter. I really like the approach grape has taken.

# <a name="decouple-frontend">Decoupling frontend from backend</a>

To be able to create many views on top of single set of APIs first thing is to decouple the API from view. Currently our frontend code is in bad shape and its not very easy to refactor.

However we have identified a few things we can do to improve the frontend code (mostly [angular](http://angularjs.org/)) and decouple it from backend code:

  - Build composable components
  - Each component contains the angular html templates, css and js in a single folder
  - We should have an angular module corresponding to each component
  - Each component must have an index.html page which can be opened see the component in action
  - Each component should add states to [ui-router](http://angular-ui.github.io/ui-router/site/#/api/ui.router) and should be loaded lazily
  - Factories should be used as primary data store for frontend and services for communication channel to upstream server


# <a name="many-more-projects">Many more projects</a>

We will focus on analytics engine, but the other projects are on road map as well.

## <a name="upgrade-mongoid">Upgrade Mongoid and dependencies</a>

We are stuck with mongoid 2.x, no test suite and a lot of breaking changes in mongoid 3.x. Mongoid 4 is going to be out soon. We want to switch to latest version of 3.x series and start preparing for the 4.x upgrade. We would need to upgrade all the other gems which are stuck die to mongoid: carrier-wave, delayed_job, rails.

## <a name="testing-framework">Testing framework</a>

A lot of our backlog is due to no test suite in place. We want to a setup a basic test suite which can break the intertia and start moving things on testing end.

## <a name="backend-modules">Component structure on backend</a>

After we started implementing the component structures for new elements on frontend we realized that it can be done in backend also. Currently we have many sub-directories in `app/`. We have jobs, services, abilities, pub_sub, assets, utilities and more. Instead we want to have three sub-directories in our app directory: views, models, modules. We will keep views as is due to view_path thingy. I also believe that mongoid checks for model declarations in `app/models` while running rake tasks like create indexes. Everything else, like controllers, mailers, services, modules, subscribers, publishers should rest in a single directory for a single module. This would have the additional benefit of each module being name spaced due to directory structure. Each module would have a init.rb which would be required in a rails initializer. This would behave very much like composable components on frontend.

# <a name="intern-call">Internship call out</a>

We are looking for interns who are keen to learn, apply, implement and see their efforts going live to users. Anyone can apply:

  - students looking for 2-3 month internships, you gotta be good to do something useful in 2-3 months
  - graduates having some time at their hands, who want ot work on cool awesome things
  - students who can do a on-site internship starting with in a few months
  - if you do not fit any category, but feel that you can contribute positively do drop us an email

You can reach us at `joinus[at]almaconnect[dot]com`, `rubish[dot]gupta[at]almaconnect[dot]com`.