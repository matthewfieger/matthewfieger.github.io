---
layout: post
title:  "Pseudo Collections in Meteor"
date:   2014-06-18 16:15:00
categories: posts me
tags: ["programming", "meteor", "basecamp insights"]
---

I am using the `select2` jQuery plugin to autofill a `textarea` from an array of `tag` values.  First, on both the client and the server, I create a new collection called `tags` and assign it to the global `Tags` variable.

	Tags = new Meteor.Collection('tags');

On the server, I publish a pseudo collection of tags to this new collection.  This pattern was inspired by the 'counts-by-room' example in the official Meteor Documentation.  I call it a pseudo collection because if you look in the database on the server, you won't see any documents in this collection.  The client only sees documents that have been reactively added via the below publish function.

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

	  return this.ready();

	});

I subscribe to the `tags` publication in my route `waitOn` hook:

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
