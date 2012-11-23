---
layout: post
title: "Text.to_speech through tts-api.com"
date: 2012-11-23 09:09
comments: true
categories:
- Ruby
- TTS
---

This extremely simple Ruby module extends the String class by adding a `to_speech` method, so `text.to_speech` will perform a request to [Text-to-Speech API](http://tts-api.com/) which produces a remote mp3 file that will be downloaded to the current directory (`curl` needed):

* `text.to_speech` &rarr; `text.mp3`

#### Usage

{% gist 4065761 tts-usage.rb %}

<!-- more -->

#### Code

{% gist 4065761 tts.rb %}
