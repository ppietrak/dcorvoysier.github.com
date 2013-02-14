---
layout: post
title: 'Pushing dashdemux upstream'
author: 'David Corvoysier'
categories: Video
tags: DASH GStreamer
published: true
---
I have created a new [enhancement bug](https://bugzilla.gnome.org/show_bug.cgi?id=690555)
 in GStreamer to push gstdashdemux into gst-plugins-bad.

<!--more-->

I have been asked by the GStreamer maintainers to merge the dashdemux
plugin with the hlsdemux code into a single 'Fragmented' plugin. This 
makes sense because the DASH demux plugin already uses some basic objects
defined in the Fragmented plugin.

I will post an update as soon as the patches are accepted.
