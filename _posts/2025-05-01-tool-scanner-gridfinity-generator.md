---
layout:     post
title:      "Tool Scanner - A Gridfinity compatible custom inserts generator"
subtitle:   "Building a mobile Gridfinity generator that generates boxes, baseplates, and custom cutouts for tools and utensils."
date:       2025-05-01 08:00:00
author:     "Jan"
header-img: "img/post-toolscanner/toolscanner_bg.jpg"
---

_Get ToolScanner for iOS at:_ [https://apps.apple.com/us/app/toolscanner-gridfinity-maker/id6745150475](https://apps.apple.com/us/app/toolscanner-gridfinity-maker/id6745150475)

_Get ToolScanner for Android at:_ coming soon :)

## Introduction

_If you're after what I've actually built skip this section and go directly to [ToolScanner](#introducing-tool-scanner)._

_And a second preface: If this app gets some traction I intend to support it in the long run and add a lot more features. If you like it you're welcome to suggest your ideas and to reach out._

I've recently bought a 3D printer and among many things one suddenly figures out can be printed I've also noticed how efficient storage can become. At first I wanted to create a box storage system that would go together like LEGOs but after a while I realized I just can't beat [Gridfinity](https://gridfinity.xyz/).

In addition to being a very efficient solution with a big community I've also seen that people have been making their own custom boxes that are meant to fit a specific tool. However from my research all of the custom tool generators were a bit complicated where you had to model the tool you wanted to cut from the box or just weren't what I was looking for.

So I've built my own.

## Introducing Tool Scanner

Tool Scanner is a mobile app that generates Gridfinity compatible baseplates, boxes and custom boxes that provide tool inserts. All you need to do is to take a picture of the tool lying on an A4 sheet of paper. Add some settings like tool depth and you get a Gridfinity compatible box that can take your custom insert.

Here's how it looks:

![tool scanner custom gridfinity inserts](/img/post-toolscanner/tool_inserts.jpg)

All of what you see was generated via ToolScanner.

## How does it work?

I intend to write some additional blog posts and open source part of this project. We'll drill down a bit further in subsequent posts about the inner workings.

But to give a hint at what's at the core: a (relatively) simple OpenCV implementation that extracts the paper from the image that is taken. Then based on the paper some processing is done to determine where the tool is and calculate its dimensions. Then a contour is computed around the tool and converted to a SVG path. That path is extruted and cut from a solid Gridfinity box to yield the final result.

While it sounds simple there's a fair amount of processing required and even now it doesn't work perfectly. Yet.

## Stay tuned

That's it for this post! I'm planning to add more features and open source pieces of the platform in the coming months. Hope you enjoy using it!
