<!-- MarkdownTOC depth=0 -->

- <!-- /MarkdownTOC -->
	- date:   2014-03-18 17:00:00
- Why PUB/SUB?
- What is PUB/SUB?
- Implementing basic publisher & subscriber; lets get it on
	- Publisher
	- Subscriber
- Problems
	- Avoid callback chain due to denormalization
- Cocluding thoughts

<!-- /MarkdownTOC -->
---
layout: post
title:  "PUB/SUB in Rails; using ActiveSupport::Notifications"
date:   2014-03-18 17:00:00
---

This article is about implementing simple PUB/SUB in Rails, using ActiveSupport::Notifications

# Why PUB/SUB?

Here at Alma Connect we are heavy users of Ruby on Rails. RoR is a fast paced web development framework preferring convention over configuration. Rails features and intial project structure encourage fat model skinny controller approach. As the project evolves maintaining it becomes a real hassel and new project structures ([interactors][1], [form objects][2], decorators[3]). Different projects have different requirements and a lot of times people end up creating there own versions of solutions to problems faced by many. Today we will talk about use of pub/sub pattern to create a more de-coupeled system.

# What is PUB/SUB?
Pub/Sub (Publisher/Subscriber) is a very basic pattern with a lot variants available out there. Lets just get on the same page here and define the pub/sub we are talking about. Most of rails developers out there will be familiar with javascript and jquery. The pub/sub architecture we are talking about is very much similar to jQuery events. If we want to capture it in bullet points:

  1. Publishers can pulish events with a string name and a payload hash
  2. Subscribers can subscribe to events either by passing a name or a regexp

# Implementing basic publisher & subscriber; lets get it on

Such a system is already bundled with rails: its [ActiveSupport::Notifications][4](ASN). ASN is used through out rails internally for instrumentation. The basic implmentation of ASN is such a simple yet powerful API that we can build our pub/sub implementation on top of it.

## Publisher

[Publisher(gist: code and usage)][5]

This is as simple as it gets. Publisher can broadcast(Publisher.broadcast_event(name, payload={}) ) an event accepting an event name and a payload hash. It can accept an optional block and will report the time taken to execute the block and also if any error occured during execution of block(this is all provided by ASN used as base, fun isn't it).

## Subscriber
[Subscriber(gist: code and usage)][6]

This is also pretty striagh forward, to subscribe( Subscriber.subscribe(name)) to an event just register with a name and the [ASN event][7] object gets passed to the attached block as the parameter.

# Problems

Basic promblems we face at Alma Connect:

  1. Avoid callback chain due to denormalization
  2. Detect and hadle events like user milestone reached, batchmate signed up 
  3. Mail delieveries coupled with code


## Avoid callback chain due to denormalization

We use mongodb and rely heavily on denormalization to cutdown on queries. We have developed an internal plugin handling denormalization of data and keeping it in sync. However there is a tight coupling between data models and denormalization. We have to define denormalization macros in our data model. So each of our model knows what data to denormalize, to and from what models. It seemed good initially, but it creates too much coupling. 

I believe rails developers are good at working with modules and classes i.e. object oriented approach rather that completely functional approach. Lets extend the pub/sub model built above to handle such use cases and add a pinch of ruby goodness.

[Publisher(gist: code and usage)]

This goes some changes to be included in a class and track the namespace. It remains backward compatible :)

[Subscriber(gist: code and usage)]: Nothing changes in here

[Publisher Base(gist: code and usage)]

We have got some 



Lets create a directory named pub_sub in our app directory to organize code neatly.

# Cocluding thoughts

A simple pattern like pub/sub can be used to write highly decoupled modules which can be upgraded/added/take out without affecting the rest of the system. A very baic implementation helps solve many of the problems any projects with medium to big codebase faces and opens up the gates to a new interesting are which we can explore in the next article. 

In this article we explored decoupling of system, hoever we have built our pub/sub on top of ASN which is originally used for instrumentation in rails. Lets get to instrumentation and see how can we use our pub/sub for instrumentations. We will also discuss about possible ways of visualizing our instrumented data and an approach to build more composable application with pub/sub.

[1][https://github.com/collectiveidea/interactor]
[2][https://github.com/apotonick/reform]
[3][https://github.com/drapergem/draper]
[4][http://api.rubyonrails.org/classes/ActiveSupport/Notifications.html]
[5][https://gist.github.com/rubish/9598373/a79e883187d2e5f92c990abed4fbb6b980776039#file-publisher-rb]
[6][https://gist.github.com/rubish/9598373/a79e883187d2e5f92c990abed4fbb6b980776039#file-subscriber-rb]
[7][http://api.rubyonrails.org/classes/ActiveSupport/Notifications/Event.html]