---
layout: page
title: Berry Battle
date: 2024-01-26 11:59:00-0000
description: A game simulator for AI controlled SBCs
img: assets/img/berry-battle-logo-white-text.png
importance: 1
category: Berry Battle
giscus_comments: false
giscus_repo: aven-arlington/flashcards-cli
toc:
  sidebar: left
---
## Background
I was recently looking around for local game jams and other ways to connect with the development community in a more social setting when I cam across a programming competition held at MIT called [Battlecode](https://battlecode.org/). The project looked amazing and I instantly tried to sign up only to discover that it is mostly limited to students. The more I thought about how I would program an AI to compete against another AI the more I wanted to try it out... but with Rust. So I have decided to make my own home grown game simulator to interface with AI running on Single Board Computers (SBC).

## What Does the Simulator Need to Accomplish?
I started writing down what I want out a meta-game like this and what seems like it would make it the most fun to tinker with. I came up with the following rough technical criteria:
- I love programming in Rust but not everyone does. So I want to have the AI node be as language agnostic as possible so it is accessible to the most people.
- The interactions with AI nodes be as safe from exploitation as possible so people will feel comfortable participating.
- Running a game should be easy so that small groups of people can setup ad-hoc tournaments or play with co-workers over the weekend etc. For this, we need automation to be a consideration from the ground up.
- Players want to learn new things when developing an AI node so the competition should encourage a high performance mindset to innovate and push the boundaries of the SBC hardware.
- The game should be fair. All AI nodes competing against each other should be running on the same SBC hardware, and the simulator should support many if not most SBCs. Note that the simulator will be created with execution on a Raspberry Pi device in mind due to their widespread availability.

## How Will it Work?
The short answer is that I don't know... yet. I have a rough idea, a napkin sketch if you will, of the high level system diagram.

<div class="row justify-content-sm-center">
    <div class="col-sm-8 mt-3 mt-md-0">
        {% include figure.html path="assets/img/physical_configuration.png" title="Physical Configuration" class="img-fluid rounded z-depth-1" %}
    </div>

</div>

In it, a gaming or server class desktop PC can act as the game simulator and will interface with the AI nodes to direct the flow of the game. My initial thoughts are to use gRPC for communications since it meet the language agnostic criteria and, from what I can discern anyway, pretty secure. The simulation engine will likely use something like Bevy to handle the graphics and execute the game loop. Bevy would then be providing updates to and requesting input from the AI nodes before rendering the game state on screen. 

The user defined AI node would need to receive the pre-defined gRPC message and then process the packaged game state data. How that is done would be entirely up to the human competitor whether is is hand coded if/else statements or a complex tensorflow model trained on a billion simulated games. Whatever the hardware supports in the execution time of a single game loop would be allowed. It would then provide the game input as a response to the gRPC message.

This first post is only intended to be a high level summary of what the project is and where I would like it to go. Initial prototyping and experimentation has already begun and hope to have a more topic centered post about it up soon.

## Additional Resources
[Berry Battle](https://www.berrybattle.com/)


