---
layout: post
title:  "Meteor OAuth with Vimeo"
date:   2014-06-25 17:00:00
categories: posts me
tags: ["programming", "meteor", "basecamp insights"]
---

This is a pattern for building OAuth bindings in Meteor.  Here, I am using OAuth to request a bunch of videos from the Vimeo Staff Pick's Channel that I can use as data fixtures when testing my app.

OAuth isn't thoroughly documented in Meteor yet and a couple of methods that I use below start with an `_`, which I think means that they are Meteor internal functions that are subject to future change.  But in the meantime, this pattern should work for standard OAuth 1.0 bindings.

For more information, I recommend checking out [Vimeo's API documentation](https://developer.vimeo.com/apis/advanced/methods/vimeo.channels.getVideos) as well as [the source for Meteor's OAuth bindings](https://github.com/meteor/meteor/blob/master/packages/oauth1/oauth1_binding.js).

	var vimeoStaffPicks = [];

	var vimeoRequest  = function (page) {

	// Set parameters
	var parameters = {
	  config : {
		// Client ID // Also known as Consumer Key or API Key
		'consumerKey' : Meteor.settings.vimeo.consumerKey,
		 // Client Secret // Also known as Consumer Secret or API Secret
		'secret' : Meteor.settings.vimeo.secret,
		// My specific access token, not needed for this request
		//'accessToken' : Meteor.settings.vimeo.accessToken,
		// My specific access token secret, not needed for this request
		//'accessTokenSecret' : Meteor.settings.vimeo.accessTokenSecret,
		'method' : 'vimeo.channels.getVideos',
		'format' : 'json',
		'channel_id' : 'staffpicks',
		'page' : page,
		'per_page' : 50,
		'oauth_signature_method' : "HMAC-SHA1",
		'oauth_version' : "1.0"
	  },
	  urls : {
		'requestToken' : "https://vimeo.com/oauth/request_token",
		'authorize' : "https://vimeo.com/oauth/authorize",
		'accessToken' : "https://vimeo.com/oauth/access_token",
		'authenticate' : "http://vimeo.com/api/rest/v2"
	  }
	}

	  // Create OAUTH1 headers
	  var oauthBinding = new OAuth1Binding(parameters.config, parameters.urls);
	  var headers = oauthBinding._buildHeader();

	  // Make request
	  var response =  oauthBinding._call(
		'GET',
		'http://vimeo.com/api/rest/v2',
		headers,
		parameters.config
	  );

	  // Do something with the results
	  // In this case, I am interested in just the 'id' of each video
	  if (response.data.stat == 'ok' && response.data.videos.video) {
		_.each(response.data.videos.video, function (element, index) {
		  vimeoStaffPicks.push(element.id);
		  console.log('New video ' + element.id + ' retrieved.');
		  return
		});
	  };

	  return response.data.stat;

	};

	var vimeoRequests = function (callback) {
	  // Request 5000 videos
	  for (var i = 0; i < 5000; i++) {
		  vimeoRequest(i);
	  }
	  callback();
	};

	vimeoRequests(function () {
	  // log the vimeoStaffPicks array to the console
	  console.log(vimeoStaffPicks);
	});
