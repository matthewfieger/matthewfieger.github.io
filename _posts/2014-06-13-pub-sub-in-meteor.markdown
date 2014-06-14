---
layout: post
title:  "Pub/Sub Patterns in Meteor"
date:   2014-06-13 16:15:00
categories: posts me
tags: ["programming", "meteor", "basecamp insights"]
---

I've been working on a project with the [MeteorJS](http://www.meteor.com) framework for about four months now.  The idea is to create a cloud-based [design-thinking](http://en.wikipedia.org/wiki/Design_thinking) platform for the enterprise called Basecamp Insights.  I don't plan on open-sourcing this app, so instead I would like to post a few Meteor patterns I have been using throughout my code.

For this first post, let's start with publications and subscriptions (pub/sub), which are the basic building blocks of any Meteor app.

	Meteor.publish(name, func)
	Meteor.subscribe(name [, arg1, arg2, ... ] [, callbacks])

In Meteor, documents are stored in what are called a collections. These the above functions control the flow of documents between the client and the server.  Just like their names would suggest, the server determines which documents are published and the client determines which documents are subscribed to.  When the client subscribes to a publication, the server sends the associated documents, and the client stores a copy of each locally.  Meteor does the behind the scenes work of keeping the local copies up to date with what is on the server.  So here are a few example patterns of pub/sub in Meteor.

This pattern only publishes documents if the client is logged in:

	// On the server
	  Meteor.publish('comments', function () {
		  if (this.userId) {
			return Comments.find();
		  }
		  return this.ready();
	  });

	// On the client
	Meteor.subscribe('comments')

This pattern only publishes documents whose 'projectId' value matches a role that is assigned to the client:

	// On the server
	  Meteor.publish('comments', function () {
		  if (this.userId) {
			var roles = Meteor.users.findOne({_id : this.userId}).roles;
			if (roles) {
				return Comments.find({projectId : {$in : roles}});
			}
		  }
		  return this.ready();
	  });

	// On the client
	Meteor.subscribe('comments')

This pattern publishes specific fields from the 'users' collection:

	// On the server
	Meteor.publish('users', function () {
	  if (this.userId) {
		var roles = Meteor.users.findOne({_id : this.userId}).roles;
		if (roles) {
			return Meteor.users.find({roles : {$in : roles}}, {fields: {username: 1, emails: 1, profile: 1, roles: 1, status: 1 }});
		}
	  }
	  return this.ready();
	});

	// On the client
	Meteor.subscribe('users')


This pattern takes a simple argument.  It uses the Underscore 'contains' method to check if the argument is in the user's 'roles' array and the publishes only documents that match that role:

	// On the server
	Meteor.publish('observations', function (arguments) {
		if (this.userId) {
			var roles = Meteor.users.findOne({_id : this.userId}).roles;
			if ( _.contains(roles, arguments.projectId) ) {
				return Observations.find({'projectId' : arguments.projectId});
			}
		}
	  return this.ready();
	});

	// On the client
	Meteor.subscribe('observations', {'projectId': Session.get("currentProject")})

This pattern can take up to two arguments, and checks for the existence of each before deciding what to do:

	// On the server
	Meteor.publish('videos', function (arguments) {
		if (!!this.userId) {
			var roles = Meteor.users.findOne({_id : this.userId}).roles;
			if (_.contains(roles, arguments.projectId)) {
				if (!!arguments.projectId && !!arguments.video_id) {
					return Videos.find({
						'projectId' : arguments.projectId,
						'video_id': arguments.video_id
					});
				}
				else if (!!arguments.projectId){
					return Videos.find({'projectId' : arguments.projectId});
				}
				else {
					return
				}
			}
		}
		return this.ready();
	});

	// On the client
	Meteor.subscribe('videos', {'projectId': Session.get("currentProject"), 'video_id': param})

This pattern individually adds documents to publish to the client.  It publishes one of each of a specific type of document:

	Meteor.publish('videosThumbnails', function () {
		var self = this
		if (!!this.userId) {
			var roles = Meteor.users.findOne({_id : this.userId}).roles;
			_.each(roles, function (element, index) {
				var project = Videos.findOne({'projectId' : element});
				self.added('videos', project._id, project);
			});
			return self.ready()
		}
		return self.ready();
	});

	// On the client
	Meteor.subscribe('videosThumbnails')

You can specify which code is executed on the client and which code is executed on the server using Meteor's built in directory structure.  Anything in the 'client' folder gets executed on the client and anything in the 'server' folder gets executed on the server.  You could also use Meteor's 'isClient' and 'isServer' methods:

	if (Meteor.isServer) {
	  Meteor.publish('comments', function () {
		  if (this.userId) {
			return Comments.find();
		  }
		  return this.ready();
	  });
	}

	if (Meteor.isClient) {
		Meteor.subscribe('comments')
	}
