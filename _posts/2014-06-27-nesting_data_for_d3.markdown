---
layout: post
title:  "Nesting Data for D3"
date:   2014-06-27 17:00:00
categories: posts me
tags: ["programming", "meteor", "d3", "basecamp insights"]
---

If you are working with D3, there is a good chance that you will want to nest data.  Getting your data into common D3 formats is often half the battle when working with D3, so you really want to have a number of tools that transform your data into these common formats.  For nesting, there are two tools that I am familiar with.  The first is `D3.nest()`.  Let's say our data is an array of objects like this:

{% highlight javascript %}
	data = [
		{'name' : 'Post 1', '0' : 'tag_a', '1' : 'tag_b'},
		{'name' : 'Post 2', '0' : 'tag_a', '1' : 'tag_c'}
	];
{% endhighlight %}

We can easily nest that using `D3.nest()` like so:

{% highlight javascript %}
	data = d3.nest()
		.key(function(d) { return d['0']; })
		.key(function(d) { return d['1']; })
		.entries(data);
{% endhighlight %}

The result is a hierarchical nest of data of `key` and `values`.  Each branch node has a `key` property specifying the name of the branch, and a `values` property specifying one or more child nodes.  Leaf nodes contain the original keys and values for each data item that was passed to the nest method.

{% highlight javascript %}
	[
	  {
		"key": "tag_a",
		"values": [
		  {
			"key": "tag_b",
			"values": [
			  {
				"0": "tag_a",
				"1": "tag_b",
				"name": "Post 1"
			  }
			]
		  },
		  {
			"key": "tag_c",
			"values": [
			  {
				"0": "tag_a",
				"1": "tag_c",
				"name": "Post 2"
			  }
			]
		  }
		]
	  }
	]
{% endhighlight %}

The second tool is `Underscore.nest()`.  In similar fashion, we can nest our data like so:

{% highlight javascript %}
	data = _.nest(data, ['0', '1']);
{% endhighlight %}

The result again is a hierarchical data structure, but this time one that is based on `name` and `children`.  Each branch node has a `name` property, specifying its name and a `children` property, specifying one or more child nodes.  Leaf nodes again contain the original keys and values for each data item.  Note that the `name` and `children` format of Underscore.nest() matches the format used in many of the D3 examples that rely on `flare.json`.

{% highlight javascript %}
	{
	  "children": [
		{
		  "name": "tag_a",
		  "children": [
			{
			  "name": "tag_b",
			  "children": [
				{
				  "0": "tag_a",
				  "1": "tag_b",
				  "name": "Post 1"
				}
			  ],
			  "index": 0
			},
			{
			  "name": "tag_c",
			  "children": [
				{
				  "0": "tag_a",
				  "1": "tag_c",
				  "name": "Post 2"
				}
			  ],
			  "index": 1
			}
		  ],
		  "index": 0
		}
	  ]
	}
{% endhighlight %}

So far so good with D3.nest() and Underscore.nest().  Let's try something else.  What if our data has arbitrary depth like so:

{% highlight javascript %}
	data = [
		{'name' : 'Post 1', '0' : 'tag_a', '1' : 'tag_b'},
		{'name' : 'Post 3', '0' : 'tag_a', '1' : 'tag_c', '2' : 'tag_b'}
	];
{% endhighlight %}

D3.nest() gives us this:

{% highlight javascript %}
	data = d3.nest()
			.key(function(d) { return d['0']; })
			.key(function(d) { return d['1']; })
			.key(function(d) { return d['2']; })
			.entries(data);

	// returns
	[
	  {
		"key": "tag_a",
		"values": [
		  {
			"key": "tag_b",
			"values": [
			  {
				"key": "undefined",
				"values": [
				  {
					"0": "tag_a",
					"1": "tag_b",
					"name": "Post 1"
				  }
				]
			  }
			]
		  },
		  {
			"key": "tag_c",
			"values": [
			  {
				"key": "tag_b",
				"values": [
				  {
					"0": "tag_a",
					"1": "tag_c",
					"2": "tag_b",
					"name": "Post 3"
				  }
				]
			  }
			]
		  }
		]
	  }
	]

{% endhighlight %}

and Underscore.nest() gives us this:

{% highlight javascript %}
	data = _.nest(data, ['0', '1', '2'])

	//returns
	{
	  "children": [
		{
		  "name": "tag_a",
		  "children": [
			{
			  "name": "tag_b",
			  "children": [
				{
				  "name": "undefined",
				  "children": [
					{
					  "0": "tag_a",
					  "1": "tag_b",
					  "name": "Post 1"
					}
				  ],
				  "index": 0
				}
			  ],
			  "index": 0
			},
			{
			  "name": "tag_c",
			  "children": [
				{
				  "name": "tag_b",
				  "children": [
					{
					  "0": "tag_a",
					  "1": "tag_c",
					  "2": "tag_b",
					  "name": "Post 3"
					}
				  ],
				  "index": 0
				}
			  ],
			  "index": 1
			}
		  ],
		  "index": 0
		}
	  ]
	}
{% endhighlight %}

Now we have a problem.  Both D3.nest() and Underscore.nest() try to access keys that are `undefined`, and go on to create branch nodes with the name `undefined`. They are both assuming that our data has uniform depth.  However we don't want undefined branches - we want the nesting to stop at leaf nodes, wherever they may be.

So, how do we nest data of arbitrary depth?  Someone pointed me towards a function called [burrow.js](https://gist.github.com/syntagmatic/4076122#file_burrow.js) that does exactly that.

In my case, I am nesting data for something similar to [D3's Zoomable Treemap Example](http://bost.ocks.org/mike/treemap/) so I ended up modifying it slightly to fit that case.

{% highlight javascript %}
	// nest data based on nodes of arbitrary depth
	// requires Underscore.js

	// 'burrow' can take an array of objects,
	// with each object item having a 'nodes' property,
	// and with the 'nodes' property containing
	// an array of node items of arbitray number.
	// or 'burrow' can take an array of arrays,
	// with each arrary containing an arbtitary number of node items.
	// 'burrow' can also take an optional,
	// root node name.
	// optionally, if you are passing an array of objects,
	// each object can also have a 'leafData' property,
	// that contains either a string or an object
	// of data to be included in the leaf nodes.
	var burrow = function (data, root) {
	  // start with a simple nested object
	  var object = {};

	  _(data).each(function(d) {
		var _object = object;
		//console.log(d);
		// nest based on an array of nodes,
		// with arbitray length,
		// meaning that we could have a branch,
		// or a leaf node at any depth
		var nodes;
		if ( _.isObject(d) && !!d.nodes ) {
		  // if d is an object,
		  // and d has nodes property,
		  // consider this the nodes
		  d['leafData'] = (!!d['leafData']) ? d['leafData'] : {};
		  nodes = d.nodes;
		}
		else if ( _.isArray(d) ) {
		  // if d is an array,
		  // consider this the nodes
		  d = {'nodes' : d, 'leafData' : {} };
		  nodes = d.nodes;
		} else {
		  // else the data is invalid
		  throw new TypeError("invalid data");
		}

		_(nodes).each(function(element,index) {
		  _object[element] = _object[element] || {'leafData' : d.leafData};
		  _object = _object[element];
		}); // _.each(d.nodes)
	  }); // _.each(data)

	  // recursively create children array
	  var descend = function (object, leafData) {
		if (!!object.leafData) {
		  // if there is leafData,
		  // let's set it aside,
		  // and delete this property from the object
		  var leafData = object.leafData;
		  delete object['leafData'];
		} else if ( !!leafData ) {
		  // if there is data that has already been set aside,
		  // let's keep that with us as we descend
		  var leafData = leafData;
		}

		// check if there are nodes at the current level,
		// by checking if the 'object' argument is empty
		if (!_.isEmpty(object)) {
		  // if there are nodes at this level,
		  // then create them and add them to the array
		  var array = [];
			_(object).each(function(value, key) {
			  if (key !== 'leafData') {
			  // check if there are any nodes,
			  // at the next level, and also
			  // pass the leafData down to the next level
			  var children = descend(value, leafData);
			  if (!!children) {
				// if there are nodes at the next level,
				// then this is a branch node,
				// and we want to recusively call the descend function again
				var node = {
				  name: key,
				  children: children
				}; // node
			  } else {
				// if there are no nodes at the next level,
				// then this is a leaf node,
				// and so it doesn't have any children,
				// and we don't have to descend anymore
				var node = {
				  name: key,
				  value: 1
				}; // node
				// and we want to include the leaf data
				// at the leaf node
				if ( !!leafData && _.isObject(leafData) ) { _.extend(node, leafData) }
				else if (!!leafData) { node['leafData'] = leafData };
			  } // else
			  array.push(node)
			} // if key
			}); // _.each
		  return array;
		} // if
		// else if there are no nodes at this level,
		// then we want to return false,
		// so that the previous level knows,
		// that there are no nodes at this level
		else return false;
	  }; // descend

	  // nested objectect
	  return {
		name: root || 'Root Node', // name of the root node
		children: descend(object)
	  }; // return
	}; // burrow


	// Exports
	if (!!_) {
	  _.mixin({"burrow" : burrow});
	} else {
	  Burrow = burrow;
	}
{% endhighlight %}

The code can be found [here](https://github.com/matthewfieger/underscore.burrow).