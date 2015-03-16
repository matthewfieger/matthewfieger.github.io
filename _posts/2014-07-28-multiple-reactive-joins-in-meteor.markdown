---
layout: post
title:  "Multiple Reactive Joins in Meteor"
date:   2014-07-28 22:48:00
categories: posts me
tags: ["programming", "meteor", "basecamp insights"]
---

I just spent the last few hours developing a new feature for [Basecamp Insights](http://basecampinsights.com).  The idea is to create a client-side activity feed that monitors activity in four different collections in real time.  I decided that the best way to go about this would be to reactively join the relevant documents from each of those four collections on the server and then publish them to a client side pseudo-collection.  Alternatively, we could send all four collections down to the client and then assemble the activity feed from there.  In that case however, we would be over-publishing documents if its just the activity feed that the client is interested in.  So with a reactive join approach, although we are asking the server to do a little more work, we are minimizing wait times for the client by only publishing exactly what is needed.

To reiterate, the server is joining four different collections into a single pseudo-collection that is published to the client.  In the code below, you will see the same patterns repeated four times.  For each of those four collections, we are essentially doing two things.

First, when the publish function first runs, for each of the four collections we:

* Call the `find` method, using a few selectors to find only a subset of documents
* Call the `forEach` method to iterate through each returned document
* Call `self.added` on each iteration in order to add that document to our pseudo-collection.

After we have added the initial documents to our pseudo-collection, we can now call the `observeChanges` method on each of those four collections so that we can update the pseudo-collection with any relevant changes.  When we see a change we update the pseudo-collection by calling `self.added` and `self.removed`.  Although this section appears at the top of the code below, we use the `initializing` variable to skip over it when the publish function first runs.  We don't want to add the same documents to our pseudo-collection twice.

Finally, when we see the `self.onStop()` event, we call `.stop()` on each of our `observeChanges` handler functions so that we stop observing changes.

On the client, we create and subscribe to the new collection with:
{% highlight javascript %}
	Activity = new Meteor.Collection('activity')
	Meteor.subscribe('activity', {'projectId': Session.get("currentProject"), limit: 100})
{% endhighlight %}

Below is the code for the publish function.  The same basic pattern should work for reactively joining an arbitrary number of collections and publishing the result to the client.  If anyone can suggest a simpler approach for multiple reactive joins in Meteor, please reach out to me on Twitter!

{% highlight javascript %}
	// server: publish an activity feed based on 4 different collections
	Meteor.publish("activity", function(arguments) {
		var self = this;
		var projectId = arguments.projectId;

		if (!!arguments.limit){
			var limit = arguments.limit;
		} else {
			var limit = 1000000;
		}

		if (this.userId) {
			var roles = Meteor.users.findOne({
				_id: this.userId
			}).roles;
			if (_.contains(roles, projectId)) {


				// we will use this to skip our
				// Observe changes during initialization
				var initializing = true;


				// have to observe changes for each collection
				// that we want to include in the activity feed

				// Observe changes in the Observations collection
				var observationsHandle = Observations.find({
						'projectId': projectId
					}, {
						fields: {
							'_id': 1,
							'submitted': 1,
							'userId': 1,
							'video_id': 1
						}
					})
					.observeChanges({
						added: function(id, fields) {
							if (!initializing) {
								var user = Meteor.users.findOne({
									_id: fields.userId
								});
								var userName = user.profile.firstName
								+ " " + user.profile.lastName;
								var videoTitle = Videos.findOne({
									'video_id': fields.video_id
								}).title;
								self.added("activity", id, {
									'projectId': projectId,
									'type': 'observation',
									'submitted': fields.submitted,
									'userName': userName,
									'videoTitle': videoTitle
								});
							}
						},
						removed: function(id) {
							self.removed("activity", id);
						}
					});

				// Observe changes in the Videos collection
				var videosHandle = Videos.find({
						'projectId': projectId
					},
					{
						fields: {
							'_id': 1,
							'submitted': 1,
							'userId': 1,
							'video_id': 1,
							'title': 1
						}
					})
			.observeChanges({
				added: function(id, fields) {
					if (!initializing) {
						var user = Meteor.users.findOne({
							_id: fields.userId
						});
						var userName = user.profile.firstName
						+ " " + user.profile.lastName;
						self.added("activity", id, {
							'projectId': projectId,
							'type': 'video',
							'submitted': fields.submitted,
							'userName': userName,
							'videoTitle': fields.title
						});
					}
				},
				removed: function(id) {
					self.removed("activity", id);
				}
			});

			// Observe changes in the Endorsements collection
			var endorsementsHandle = Endorsements.find({
					'projectId': projectId
				}, {
					fields: {
						'_id': 1,
						'submitted': 1,
						'userId': 1,
						'ref': 1,
						'refType': 1
					}
				})
				.observeChanges({
					added: function(id, fields) {
						if (!initializing) {
							var user = Meteor.users.findOne({
								_id: fields.userId
							});
							var userName = user.profile.firstName
							+ " " + user.profile.lastName;
							if (fields.refType === 'video') {
								var videoTitle = Videos.findOne({
									'_id': fields.ref
								}).title;
							} else if (fields.refType === 'observation') {
								var videoReference = Observations.findOne({
									'_id': fields.ref
								}).video_id;
								var videoTitle = Videos.findOne({
									'video_id': videoReference
								}).title;
							}
							self.added("activity", id, {
								'projectId': projectId,
								'type': 'endorsement',
								'submitted': fields.submitted,
								'userName': userName,
								'videoTitle': videoTitle
							});
						}
					},
					removed: function(id) {
						self.removed("activity", id);
					}
				});

			// Observe changes in the Comments collection
			var commentsHandle = Comments.find({
					'projectId': projectId
				}, {
					fields: {
						'_id': 1,
						'submitted': 1,
						'userId': 1,
						'ref': 1,
						'refType': 1
					}
				})
				.observeChanges({
					added: function(id, fields) {
						if (!initializing) {
							var user = Meteor.users.findOne({
								_id: fields.userId
							});
							var userName = user.profile.firstName
							+ " " + user.profile.lastName;
							if (fields.refType === 'video') {
								var videoTitle = Videos.findOne({
									'_id': fields.ref
								}).title;
							} else if (fields.refType === 'observation') {
								var videoReference = Observations.findOne({
									'_id': fields.ref
								}).video_id;
								var videoTitle = Videos.findOne({
									'video_id': videoReference
								}).title;
							}
							self.added("activity", id, {
								'projectId': projectId,
								'type': 'comment',
								'submitted': fields.submitted,
								'userName': userName,
								'videoTitle': videoTitle
							});
						}
					},
					removed: function(id) {
						self.removed("activity", id);
					}
				});

			initializing = false;

			//Observations Initial Activity
			Observations.find({
					'projectId': projectId
				}, {
					fields: {
						'_id': 1,
						'submitted': 1,
						'userId': 1,
						'video_id': 1
					},
					limit: limit,
					sort: {submitted: -1}
				})
				.forEach(function(observation) {
					var user = Meteor.users.findOne({
						_id: observation.userId
					});
					var userName = user.profile.firstName
					+ " " + user.profile.lastName;
					var videoTitle = Videos.findOne({
						'video_id': observation.video_id
					}).title;
					self.added("activity", observation._id, {
						'projectId': projectId,
						'type': 'observation',
						'submitted': observation.submitted,
						'userName': userName,
						'videoTitle': videoTitle
					});
				});

			//Videos Initial Activity
			Videos.find({
					'projectId': projectId
				}, {
					fields: {
						'_id': 1,
						'submitted': 1,
						'userId': 1,
						'video_id': 1
					},
					limit: limit,
					sort: {submitted: -1}
				})
				.forEach(function(video) {
					var user = Meteor.users.findOne({
						_id: video.userId
					});
					var userName = user.profile.firstName
					+ " " + user.profile.lastName;
					var videoTitle = Videos.findOne({
						'video_id': video.video_id
					}).title;
					self.added("activity", video._id, {
						'projectId': projectId,
						'type': 'video',
						'submitted': video.submitted,
						'userName': userName,
						'videoTitle': videoTitle
					});
				});

			//Endorsements Initial Activity
			Endorsements.find({
					'projectId': projectId
				}, {
					fields: {
						'_id': 1,
						'submitted': 1,
						'userId': 1,
						'ref': 1,
						'refType': 1
					},
					limit: limit,
					sort: {submitted: -1}
				})
				.forEach(function(endorsement) {
					var user = Meteor.users.findOne({
						_id: endorsement.userId
					});
					var userName = user.profile.firstName
					+ " " + user.profile.lastName;
					if (endorsement.refType === 'video') {
						var videoTitle = Videos.findOne({
							'_id': endorsement.ref
						}).title;
					} else if (endorsement.refType === 'observation') {
						var videoReference = Observations.findOne({
							'_id': endorsement.ref
						}).video_id;
						var videoTitle = Videos.findOne({
							'video_id': videoReference
						}).title;
					}

					self.added("activity", endorsement._id, {
						'projectId': projectId,
						'type': 'endorsement',
						'submitted': endorsement.submitted,
						'userName': userName,
						'videoTitle': videoTitle
					});
				});

			//Comments Initial Activity
			Comments.find({
					'projectId': projectId
				}, {
					fields: {
						'_id': 1,
						'submitted': 1,
						'userId': 1,
						'ref': 1,
						'refType': 1
					},
					limit: limit,
					sort: {submitted: -1}
				})
				.forEach(function(comment) {
					var user = Meteor.users.findOne({
						_id: comment.userId
					});
					var userName = user.profile.firstName
					+ " " + user.profile.lastName;
					if (comment.refType === 'video') {
						var videoTitle = Videos.findOne({
							'_id': comment.ref
						}).title;
					} else if (comment.refType === 'observation') {
						var videoReference = Observations.findOne({
							'_id': comment.ref
						}).video_id;
						var videoTitle = Videos.findOne({
							'video_id': videoReference
						}).title;
					}

					self.added("activity", comment._id, {
						'projectId': projectId,
						'type': 'comment',
						'submitted': comment.submitted,
						'userName': userName,
						'videoTitle': videoTitle
					});
				});

			self.ready();

			self.onStop(function() {
				observationsHandle.stop();
				videosHandle.stop();
				endorsementsHandle.stop();
				commentsHandle.stop();
			});

		} //if _.contains
	} // if userId

	return self.ready();

	});
{% endhighlight %}