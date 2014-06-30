---
layout: post
title:  "Directory Structure in Meteor"
date:   2014-06-13 16:46:00
categories: posts me
tags: ["programming", "meteor", "basecamp insights"]
---

To organize my Meteor directory for Basecamp Insights, I have been using a scaffolding tool called ["em"](https://github.com/EventedMind/em).  Here is what my directory looks like to date:

	.
	├── both
	│   ├── collections
	│   ├── controllers
	│   ├── methods
	│   └── router
	├── client
	│   ├── collections
	│   ├── config
	│   ├── controllers
	│   ├── helpers
	│   ├── lib
	│   │   ├── javascripts
	│   │   │   ├── 0_bootstrap
	│   │   │   ├── 1_plugins
	│   │   │   └── 2_template
	│   │   └── stylesheets
	│   │       ├── 0_bootstrap
	│   │       ├── 1_fonts
	│   │       └── 2_template
	│   └── views
	│       ├── 0_layouts
	│       │   ├── application_layout
	│       │   ├── home_layout
	│       │   └── master_layout
	│       ├── 1_shared
	│       │   ├── loading
	│       │   └── not_found
	│       ├── 2_home
	│       │   ├── pages
	│       │   │   ├── contact
	│       │   │   ├── faq
	│       │   │   ├── features
	│       │   │   └── landing
	│       │   └── shared
	│       │       ├── footer
	│       │       ├── header
	│       │       └── main_nav
	│       └── 3_application
	│           ├── pages
	│           │   ├── 01_observations
	│           │   │   ├── index
	│           │   │   └── item
	│           │   ├── 02_insight
	│           │   ├── 03_ideas
	│           │   ├── 04_action
	│           │   ├── notifications
	│           │   ├── profile
	│           │   └── settings
	│           └── shared
	│               ├── footer
	│               ├── header
	│               ├── main_nav
	│               └── project_switcher
	├── config
	│   └── development
	├── packages
	├── public
	│   ├── fonts
	│   └── img
	├── scripts
	├── server
	│   ├── collections
	│   ├── config
	│   ├── controllers
	│   ├── db
	│   ├── lib
	│   ├── methods
	│   └── publish
	└── tests

P.S. You can easily print a directory tree like the one above using the `tree` command in OSX.  Install this command via Homebrew with `brew install tree`.
