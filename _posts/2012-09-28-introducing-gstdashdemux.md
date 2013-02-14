---
layout: post
title: 'Introducing gstdashdemux'
author: 'David Corvoysier'
categories: Video
tags: DASH GStreamer
published: true
---
[Orange](http://www.orange.com) and [ST MicroElectronics](http://www.st.com)
have joined their forces to develop an Open Source implementation of the
[MPEG-DASH](http://mpeg.chiariglione.org/standards/mpeg-dash) standard 
based on [GStreamer](http://gstreamer.freedesktop.org/).

<!--more-->

[gstdashdemux](https://github.com/Orange-OpenSource/gstdashdemux) is a
GStreamer plugin allowing the playback of MPEG-DASH streams. It is
compatible with both the legacy 0.10.x and new 1.0.x GStreamer frameworks.

The dashdemux source code is available under an LGPL V2.1 license.

## About DASH

Dynamic Adaptive Streaming over HTTP (DASH), aka MPEG-DASH, is a 
streaming techonology similar to Apple's HTTP Live Streaming (HLS) or
Microsoft's Smooth Streaming solutions.

In a nutshell, DASH works by breaking a multimedia content into a sequence
 of small "chunks" that are made available at a variety of different bit
 rates. 
 
During playback, it is up to the DASH client to select the best
alternative to download the next segment based on current network 
conditions, ie the segment with the highest possible bitrate that can be
 downloaded in time without causing stalls or rebuffering.
 
MPEG-DASH technology was developed under [MPEG](http://mpeg.chiariglione.org/).

The MPEG-DASH standard was published as 
[ISO/IEC 23009-1:2012]([http://www.iso.org/iso/iso_catalogue/catalogue_tc/catalogue_detail.htm?csnumber=57623)
in April, 2012.

## About GStreamer

[GStreamer](http://gstreamer.freedesktop.org/) is a framework for 
creating streaming media applications using plugins.

The plugins can be linked and arranged in a pipeline that defines the 
flow of the data.

The GStreamer core function is to provide a framework for plugins, 
data flow and media type handling/negotiation.

It also provides an API to write applications using the various plugins.

GStreamer has been ported to a wide range of operating systems, including
but not limited to:

- Linux on x86 and ARM,
- MacOSX,
- Microsoft Windows (using MS Visual Developer).

It is in particular widely supported on embedded platforms.

## GStreamer Implementation notes
 
The following section describes how the DASH demux GStreamer plugin
works internally.

Please refer to the [Links](#links) section to see how to get access to 
the latest version of the plugin source code.
 
### Introduction
 
dashdemux is a "fake" GStreamer demux, as unlike typical demux elements,
 it doesn't split data streams contained in an enveloppe to expose them
to downstream decoding elements.

Instead, it parses an XML file called a manifest to identify a set of
individual stream fragments it needs to fetch and expose to the actual
demux elements that will handle them (this behavior is sometimes 
referred in the GStreamer documentation as the "demux after a demux" 
scenario).

For a given section of content, several representations corresponding
to different bitrates may be available: dashdemux will select the most
appropriate representation based on local conditions (typically the 
available bandwidth and the amount of buffering available, capped by
a maximum allowed bitrate). 

The representation selection algorithm can be configured using
specific properties:

- max bitrate, 
- min/max buffering,
- bandwidth ratio.
 
The plugin is based on some basic objects defined in the GStreamer HLS
Demux plugin from the gst-plugins-bad module.

### General Design
 
dashdemux has a single sink pad that accepts the data corresponding 
to the manifest, typically fetched from an HTTP or file source.

dashdemux exposes the streams it recreates based on the fragments it
fetches through dedicated src pads corresponding to the caps of the
fragments container (ISOBMFF/MP4 or MPEG2TS).

During playback, new representations will typically be exposed as a
new set of pads (see 'Switching between representations' below).

Fragments downloading is performed using a dedicated task that fills
an internal queue. Another task is in charge of popping fragments
from the queue and pushing them downstream.
 
### Switching between representations
 
Decodebin supports scenarios allowing to seamlessly switch from one 
stream to another inside the same "decoding chain".

To achieve that, it combines the elements it autoplugged in chains
and groups, allowing only one decoding group to be active at a given
time for a given chain.

A chain can signal decodebin that it is complete by sending a 
no-more-pads event, but even after that new pads can be added to
create new subgroups, providing that a new no-more-pads event is sent.

We take advantage of that to dynamically create a new decoding group
in order to select a different representation during playback.

Typically, assuming that each fragment contains both audio and video,
the following tree would be created:
 
<pre>
chain "DASH Demux"
|_ group "Representation set 1"
|   |_ chain "Qt Demux 0"
|       |_ group "Stream 0"
|           |_ chain "H264"
|           |_ chain "AAC"
|_ group "Representation set 2"
 |_ chain "Qt Demux 1"
     |_ group "Stream 1"
         |_ chain "H264"
         |_ chain "AAC"
</pre>
 
Or, if audio and video are contained in separate fragments:
 
<pre>
chain "DASH Demux"
|_ group "Representation set 1"
|   |_ chain "Qt Demux 0"
|   |   |_ group "Stream 0"
|   |       |_ chain "H264"
|   |_ chain "Qt Demux 1"
|       |_ group "Stream 1"
|           |_ chain "AAC" 
|_ group "Representation set 2"
 |_ chain "Qt Demux 3"
 |   |_ group "Stream 2"
 |       |_ chain "H264"
 |_ chain "Qt Demux 4"
     |_ group "Stream 3"
         |_ chain "AAC" 
</pre>
 
In both cases, when switching from Set 1 to Set 2 an EOS is sent on
each end pad corresponding to Rep 0, triggering the "drain" state to
propagate upstream.
Once both EOS have been processed, the "Set 1" group is completely
drained, and decodebin2 will switch to the "Set 2" group.

Note: nothing can be pushed to the new decoding group before the 
old one has been drained, which means that in order to be able to 
adapt quickly to bandwidth changes, we will not be able to rely
on downstream buffering, and will instead manage an internal queue.

## Links

* [Dash Demux original repository on github](https://github.com/Orange-OpenSource/gstdashdemux)
* [GStreamer bug to integrate gstdashdemux in gst-plugins-bad](https://bugzilla.gnome.org/show_bug.cgi?id=690555)
