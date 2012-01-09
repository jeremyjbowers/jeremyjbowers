---
layout: post
title: "A brief lesson on configuring Varnish"
date: 2011-12-30 21:14
comments: true
categories: [Lessons, Varnish, Infrastructure]
---

It's election season, so there's no better time to get to work on your infrastructure. At the Post, we've been giving our infrastructure a bit of an early spring cleaning. 

If you're going to survive election traffic, you're going to need caching. I recommend Varnish, a slick and speedy reverse-proxy cache which sits in front of your application server and shields your app from the ravening hordes of clients. In this lesson, we're going to build out a basic Varnish config and then layer in handy things like URL priming, URL purging via regular expression, and some tricks to keep your site performing well even when you have heavy pages that need to be forcibly expired often.

Ready? Let's get cracking.
<!-- more -->
There are excellent instructions for [installing](https://www.varnish-cache.org/docs/3.0/installation/install.html) and [running](https://www.varnish-cache.org/docs/3.0/tutorial/index.html) Varnish on [their Web site](https://www.varnish-cache.org/). There are also several types of [fairly](https://www.varnish-cache.org/docs/trunk/tutorial/handling_misbehaving_servers.html) [useful](https://www.varnish-cache.org/trac/wiki/Performance) [documentation](https://www.varnish-cache.org/trac/wiki/VCLExampleGrace). 

This blog post won't focus on those things. Instead, I'm going to focus on the stuff I think is missing. The first thing I want to explain is how to build a Varnish configuration file.

Varnish Configuration
----------------------

The Varnish configuration file uses a series of subroutines ("sub") that trace a request through two paths: One path for cache misses, and one path for cache hits. Here's the basic pattern:

**Miss**: sub vcl\_recv > sub vcl\_miss > sub vcl\_fetch > sub vcl\_deliver

**Hit**: sub vcl\_recv > sub vcl\_hit > sub vcl\_deliver

A complete Varnish configuration file will need to contain directions for both of these paths. This is a bare-bones VCL, commented for your viewing pleasure.
{% gist 1542949 %}

Here's a much more complex VCL with support for PURGE, PRIME and some tricks for jQuery AJAX and parsing JSON.
{% gist 1542992 %}

This sums up almost everything I had to learn about configuring Varnish this week, especially the bits about jQuery and grace mode.