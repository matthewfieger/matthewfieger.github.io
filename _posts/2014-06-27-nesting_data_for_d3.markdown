---
layout: post
title:  "Nesting Data for D3"
date:   2014-06-27 17:00:00
categories: posts me
tags: ["programming", "meteor", "d3", "basecamp insights"]
---

If you are working with D3, there is a good chance that you will want to nest some data.  There are at least two tools that I know of to do this d3.nest and underscore.nest.  They work very nicely, except when you are dealing with data of arbitray depth.

We really need to provide a bunch of functions like this, and like you did for graph data structures, to transform our data into common d3 patterns.

The problem is this: Both d3.nest and _.nest try to access keys that
are undefined, and creating nodes with the name "undefined". I want
not to create undefined nodes. The nesting should stop at leaf nodes.

I have a data structure that looks like this below and I want to nest it to make a D3 treemap. I am able to nest it using either Underscore.nest or D3.nest(). However, because my data has arbitrary depth, I end up with a bunch of undefined nodes. How can I nest it to arbitrary depth or rollup the undefined nodes?


http://bl.ocks.org/syntagmatic/4076122#burrow.js

https://gist.github.com/syntagmatic/4076122#file_burrow.js

Both underscore.nest and d3.nest work very nicely if your data has uniform depth.

Let's say we have a collection of posts that each have an associated list of tags.  We want to nest this into a D3 format.

{% highlight javascript %}
	// Example Data
	data = [
		{'name' : 'Post 1', '0' : 'tag_a'},
		{'name' : 'Post 2', '0' : 'tag_b'},
		{'name' : 'Post 3', '0' : 'tag_b', '1' : 'tag_c'},
		{'name' : 'Post 4', '0' : 'tag_d', '1' : 'tag_e', '2' : 'tag_f', '3' : 'tag_g'}
	];
{% endhighlight %}

{% highlight javascript %}
	// Nesting with the Underscore nest plugin
	var underscoreNested = _.nest(data, ['0', '1', '2', '3']);

	{
	  "children": [
		{
		  "name": "tag_a",
		  "children": [
			{
			  "name": "undefined",
			  "children": [
				{
				  "name": "undefined",
				  "children": [
					{
					  "name": "undefined",
					  "children": [
						{
						  "0": "tag_a",
						  "name": "Post 1"
						}
					  ],
					  "index": 0
					}
				  ],
				  "index": 0
				}
			  ],
			  "index": 0
			}
		  ],
		  "index": 0
		}...

{% endhighlight %}



{% highlight javascript %}
	// Nesting with D3
	var d3Nested = d3.nest()
		.key(function(d) { return d['0']; })
		.key(function(d) { return d['1']; })
		.key(function(d) { return d['2']; })
		.key(function(d) { return d['3']; })
		.entries(data);

	[
	  {
		"key": "tag_a",
		"values": [
		  {
			"key": "undefined",
			"values": [
			  {
				"key": "undefined",
				"values": [
				  {
					"key": "undefined",
					"values": [
					  {
						"0": "tag_a",
						"name": "Post 1"
					  }
					]
				  }
				]
			  }
			]
		  }
		]
	  }...
{% endhighlight %}


I ended up modifying it slightly so that it doesn't return "children : []" at the leaves.  It works very nicely with the zoomable treemap example.
http://bost.ocks.org/mike/treemap/

{% highlight javascript %}
	// nest rows with keys array, requires Underscore.js
	burrow = function (data) {
	  // create simple nested object
	  var object = {};
	  _(data).each(function(d) {
		var _object = object;
		// branch based on an array of items,
		// with arbitray depth/length,
		// in this case the tags array
		_(d.tags).each(function(element,index) {
		  _object[element] = _object[element] || {};
		  _object = _object[element];
		});
	  });

	  // recursively create children array
	  function descend(object) {
		if (!_.isEmpty(object)) {
		  var array = [];
			_(object).each(function(value, key) {
			  var children = descend(value);
			  if (!!children) {
			    // if there are going to be children,
			    // we would like to do one thing
				var branch = {
				  name: key,
				  children: children
				};
			  } else {
			    // and if there are no children,
			    // we would like to do something else
				var branch = {
				  name: key,
				  value: 1
				};
			  }
			  array.push(branch)
			}); // _.each
			return array;
		} // if
		else return false;
	  }; // descend

	  // nested objectect
	  return {
		name: "Nested Data",
		children: descend(object)
	  };
	};
{% endhighlight %}
