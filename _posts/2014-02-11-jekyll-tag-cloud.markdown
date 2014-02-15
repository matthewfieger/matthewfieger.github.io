---
layout: post
title:  "Jekyll Tag Cloud"
date:   2014-02-11 18:30:00
categories: posts me
tags: ["programming", "jekyll"]
source:
reference:
---

[Tag clouds](http://en.wikipedia.org/wiki/Tag_cloud) can be a useful way of depicting keywords and their frequency of appearance on a website.  I thought it would be a good idea to create one for this site so that I can visualize which topics I have been interested in.

I've built the site with Jekyll, and there are at least a few [Jekyll plugins](http://jekyllrb.com/docs/plugins/) available for creating tag clouds.  The challenge is that I host the site on Github Pages, which doesn't support Jekyll plugins.  Luckily, it turns out that a clean and simple solution can be built using Jekyll's standard capabilities. [\[ 1 \]](http://vvv.tobiassjosten.net/jekyll/jekyll-tag-cloud/)

Jekyll traverses the site looking for files to process. Any files with YAML Front Matter are subject to processing. For each of these files, Jekyll makes a variety of data available via the Liquid templating system. [\[ 2 \]](http://jekyllrb.com/docs/variables/)

To create a tag cloud, the first thing to do is access the global tags variable, {% raw %}{{ site.tags }}{% endraw %}.  This returns an object of key-value pairs.  Each key represents a tag, and each key's corresponding value is an array containing all posts that cite that tag.  It can be thought of as something like this:

	{
	tagName : [post, post, post],
	tagName : [post, post, post],
	}

What we want to do is assign each tag a weighting based on the number of posts it has been cited in.  We start by counting the number of times each tag has appeared.  We can do this by iterating through each of these key-value pairs, accessing the array of posts, and then counting its size.  This returns a list with the number of times each tag has been cited:

	{% raw %}
	{% for tag in site.tags %}
	  {{ tag | last | size }}
	{% endfor %}
	{% endraw %}

Next, we take the number of times a tag has been cited and divide by the total number of tags.  Depending on the total number of tags, you may need to normalize the weighting by multiplying by an arbitrary number, such as 1000.  We can use Liquid's built in math filters to do this:

	{% raw %}
	{{ tag | last | size | times: 1000 | divided_by: site.tags.size }}
	{% endraw %}

For this tag cloud, we want the font-size of the tag to correspond to its weighting.
We can  do this with some in-line styling.  An arbitrary number (100) can also be added to the weighting in order to control the relative font-size.  The last thing that can be done is to replace spaces in tags that have more than one word with dashes, so that they aren't mistaken as separate tags.  Here is the full [implementation]({{ site.url }}/about.html):

	{% raw %}
	{% for tag in site.tags %}
	  <span style="font-size: {{ tag | last | size | times: 1000 | divided_by: site.tags.size | plus: 100 }}%">
	    {{ tag | first | replace: ' ', '-'}}
	  </span>
	{% endfor %}
	{% endraw %}

It looks like Jekyll displays the tags in reverse chronological order, as they were first cited on the blog.  As a next step, I would like to see if Jekyll can randomly sort the order in which the tags will be displayed.