---
layout:     post
title:      "Jarvis miniseries part III: Wake word activation"
subtitle:   "Using wake word detection to respond to speech for our AI-powered home assistant in Rust"
date:       2024-07-01 20:00:00
author:     "Jan"
header-img: "img/post-lanedetector/bg.jpg"
---

# Where we left off

Previously we handled VAD chunking so that we get discrete blocks of audio data. Those blocks aren't silence due to the VAD detector and hopefully contain speech.

## Goal of this post

Before we fire up AI to start doing speech to text recognition we want to be sure that the potential speech in the audio frame starts with the wake word. We obviously could use a fully fledged speech to text recognizer (which we still will) for this but that operation is relatively slow and computationally expensive. The goal of this post is to prepare a wake word detector that is fast.

<a href="/img/post-jarvis/jarvis_processing_pipeline_wakeword.png">
![Image of the processing pipeline](/img/post-jarvis/jarvis_processing_pipeline_wakeword.png)
</a>