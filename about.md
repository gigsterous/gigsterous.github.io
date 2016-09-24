---
layout: page
title: About
permalink: /about/
---

<head>
	<style media="screen" type="text/css">
	* {
  		-webkit-box-sizing: border-box;
  		-moz-box-sizing: border-box;
  		box-sizing: border-box;
  		margin: 0;
  		padding: 0;
	}

	img {
	  display: block;
	}

	#alignment {
		width:100%;
		margin:0 auto;
	  	overflow:hidden;
	  	padding:0;

	  	position: relative;
	}

	#alignment img{
		width:50%;
		max-width: 250px;
		max-height: 250px;
	  	float:right;
	}

	#alignment p { 
		width:50%;
	  	text-align:left;
	  	vertical-align: middle;
	  	float:left;
	  	padding:0;

    	position: absolute;
  		top: 50%;
  		transform: translateY(-50%);
	}
	</style>
</head>

## Gigsterous

Gigsterous is a project aiming to connect musicians with other musicians, producers or simply people seeking music for their own purpose. We believe we can help musicians connect through modern technology so that they can focus on their performance, rather than struggling to find their next gig.

We write posts for this blog about things we are learning as we develop the [final product](https://github.com/gigsterous/). Some posts are tightly related to what we do and some are examples derived from what we did to make sure you can reproduce it yourself and learn just like we did. We believe that knowledge shared is knowledge well spent.

## Team

We are two recent graduates of [Czech Technical University in Prague](http://www.fel.cvut.cz/en/) and big music enthusiasts.


### Martin

<div id="alignment">
<p>Software engineer by day, bass player by night. Currently I focus on Java related technologies and am trying to build the REST API.</p>

<img src="/assets/about/martin.png"/>
</div>

### Michal

<div id="alignment">
<p>I've got mad skillz in violin, iOS and Java. Right now, I'm all about that app and making sure it's perfect for a musician just like me.</p>

<img src="/assets/about/michal.png"/>
</div>

## Blog Engine

The blog runs on Jekyll, which you can find at
{% include icon-github.html username="jekyll" %} /
[jekyll](https://github.com/jekyll/jekyll)

The theme is a base Jekyll theme called minima. You can find the source code for it at:
{% include icon-github.html username="jekyll" %} /
[minima](https://github.com/jekyll/minima)
