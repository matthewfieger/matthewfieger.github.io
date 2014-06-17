---
layout: post
title:  "Loading Templates in Meteor's Iron Router"
date:   2014-06-14 16:15:00
categories: posts me
tags: ["programming", "meteor", "basecamp insights"]
---

[Iron Router](https://github.com/EventedMind/iron-router) is a popular routing package for Meteor.  Since the package is still under active development, the documentation is not always up to date.  One area where the documentation is currently lacking is with the use of loading templates.

Iron Router supposedly provides a global setting to specify a loading template like so:

	Router.configure({
	  layoutTemplate: 'masterLayout',
	  notFoundTemplate: 'notFound',
	  loadingTemplate: 'loading',
	});

The above configuration will give you a `layoutTemplate` and a `notFoundTemplate` but it will not give you a `loadingTemplate`.  The loading template is supposed to render while your route is waiting on subscriptions that are provided in the `waitOn` hook.  In older versions of Iron Router, this worked.  However, as of the latest release (7.0) the `waitOn` hook doesn't actually wait.  It merely provides a `this.ready()` callback and allows the rest of the route to render immediately.  When `this.ready()` returns true, the route re-renders with the updated subscriptions that you were waiting on.  The result is that the loading template never actually gets a chance to make an appearance.

So, if you want a loading template in the latest version of Iron Router, you need to actually specify that the route should not render until `this.ready()` returns true, and that the loading template should be rendered in the meantime.

Below is a route following this pattern.  My route waits on an array of subscriptions:

	   this.route('observationItem', {
		path: '/observations/:video_id',
		controller: applicationController,
		waitOn: function() {
		  var param = +this.params.video_id;
		  return [
		  	Meteor.subscribe('videos',
		  		{
		  		'projectId': Session.get("currentProject"),
		  		'video_id': param
		  		}
		  	),
		  	Meteor.subscribe('observations',
		  		{'projectId': Session.get("currentProject")}
		  	),
		  	Meteor.subscribe('endorsements'),
		  	Meteor.subscribe('comments')
		  ];
		},
		onBeforeAction: function () {
		  var param = +this.params.video_id;
		  return []
		},
		data: function () {
		  var param, query;
		  param = +this.params.video_id;
		  query = Videos.findOne({video_id: param});
		  return query;
		}
	  });

Here is this route's controller.  In the `onBeforeAction` hook, it renders my loading template.  In the `action` hook, it again renders my loading template until `this.ready()` returns true.

	applicationController = RouteController.extend({

	  layoutTemplate: 'applicationLayout',
	  notFoundTemlplate: 'notFound',
	  yieldTemplates: {
		'applicationHeader': {to: 'header'},
		'projectSwitcher': {to: 'projectSwitcher'},
		'applicationMainNav': {to: 'mainNav'},
		'applicationFooter': {to: 'footer'}
	  },
	  waitOn: function() {
		return [
			Meteor.subscribe('projects'),
			Meteor.subscribe('videosThumbnails'),
			Meteor.subscribe('users')
		];
	  },
	  onBeforeAction: function(pause) {
		this.render('loading');
		if ( !Meteor.loggingIn() || !Meteor.user() )
			{AccountsEntry.signInRequired(this)};
		if (!!this.ready() && Projects.find().count() == 0)
			{this.redirect('/dashboard')}
	  },
	  action: function () {
		if (!this.ready()) {
		  this.render('loading');
		}
		else {
		  this.render();

		}
	  }
	});

The above patterns will render the loading template until all of the subscriptions specified in the `waitOn` hook are ready.  If you want to subscribe to a publication but don't want the loading template to wait on it, you could subscribe to it in the `onBeforeAction` hook.