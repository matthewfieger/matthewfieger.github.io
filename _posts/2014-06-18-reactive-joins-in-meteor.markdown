---
layout: post
title:  "Pseudo Collections in Meteor"
date:   2014-06-18 16:15:00
categories: posts me
tags: ["programming", "meteor", "basecamp insights"]
---

Meteor offers a lot of flexibility in reactively joining and publishing data to the client.  To give an example of a reactive join, this post shows how I am using the `select2` jQuery plugin to autofill a `textarea` from an array of `tag` values.  First you will want to add the `select2` plugin:

	$ mrt add select2

Now, on both the client and the server, I create a new collection called `tags` and assign it to the global `Tags` variable.

	Tags = new Meteor.Collection('tags');

On the server, I publish a pseudo collection of tag items to this new collection.  This pattern was inspired by the 'counts-by-room' example in the official Meteor Documentation.  I refer to it as a pseudo collection because if you look in the `tags` collection on the server, you won't see any documents in there.  The client only sees documents that have been reactively added to the `tags` collection via the below publish function.

In this publish function I am first checking the arguments against the current user's roles.  Next, I am pulling in data from the `observations` collection and calling the `fetch` method on it to turn the result into an array.  I use the `pluck`, `flatten`, and `uniq` functions from Underscore to create an array of each tag item.  Down below, I take this `tags` array, and add it to the publish function with the `self.added` method.

Now that we have done that, I also want to monitor any changes to the `observations` collection so that we reactively update the publication with any new tags.  This is what is happening in `handle`.  I am observing the `observations` collection and anytime a tag is added or removed, I call `self.added` or `self.changed` to let the publish function know to update the client.  I use the `initializing` check to skip over this the first time so that we don't initially publish the same tags twice.  We return `handle.stop()` on `self.onStop()` so that we can stop observing changes when the publish function stops.  And finally, `self.ready()` lets the client know that the publication has completed.

	Meteor.publish("tags", function (arguments) {
	  var self = this;

		if (this.userId) {
			var roles = Meteor.users.findOne({_id : this.userId}).roles;
			if ( _.contains(roles, arguments.projectId) ) {

					var observations, tags, initializing, projectId;
					initializing = true;
					projectId = arguments.projectId;
					observations = Observations.find(
						{'projectId' : projectId},
						{fields: {tags: 1}}
					).fetch();
					tags = _.pluck(observations, 'tags');
					tags = _.flatten(tags);
					tags = _.uniq(tags);

					var handle = Observations.find(
						{'projectId': projectId},
						{fields : {'tags' : 1}}
					).observeChanges({
						added: function (id, fields) {
						  if (!initializing) {
							tags = _.union(tags, fields.tags);
							self.changed(
								"tags",
								projectId,
								{'projectId': projectId,
								'tags': tags}
							);
						  }
						},
						removed: function (id) {
						  self.changed(
						  	"tags",
						  	projectId,
						  	{'projectId': projectId,
						  	'tags': tags}
						  );
						}
					});

					initializing = false;
					self.added(
						"tags",
						projectId,
						{'projectId': projectId, 'tags': tags}
					);
					self.ready();

					self.onStop(function () {
						handle.stop();
					});

			} //if _.contains
		} // if userId

	  return self.ready();

	});

I subscribe to the `tags` publication in my route's `waitOn` hook:

	Meteor.subscribe('tags', {'projectId': Session.get("currentProject")})

Now we have our tags available and ready to work with on the client.  We initialize `select2` in a template helper function.  We get reactivity by placing it inside `Deps.autorun`. Whenever a new tag is added by one client, it is reactively published to all other relevant clients, giving them near immediate access to it.

	Template.observationPostForm.rendered = function () {
		var self = this;
		self.myDeps = Deps.autorun(function () {
			var cursor = Tags.findOne({'projectId': Session.get("currentProject")})
			if (cursor && cursor.tags) {
				var options = {
				  'multiple': true,
				  'selectOnBlur': true,
				  'tags': cursor.tags,
				  'tokenSeparators': [","],
				  'width': 'element'
				};
				self.$('#post-tags').select2(options);
			}
		});
	};

And finally here is the relevant code in my template:

	<textarea id="post-tags" name="post-tags" placeholder="Add comma separated tags..." rows="1" required >
	</textarea>
