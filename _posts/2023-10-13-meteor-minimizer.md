---
title: Meteor Minimizer - A Game Jam Game in Retrospect
tags: TeXt
---

Meteor Minimizer was a short game that I made during Game Boy Jam 11 in September of 2023 using the Godot engine.
From the beginning, I sought to implement patterns that Godot supports well - particularly the Observer Pattern.
I found that the pattern was about as strong as I expected - and that was very strong. I employed a singleton (and the singleton pattern) to preload various signals to call or be called on across other classes. I used this for events that, when occuring, various classes would interact with their output - as an example, a meteor being shot by the player's bullet, reducing its hp to 0 would cause a number of events across classes to occur. The point counter would increase, audio would play, we'd check if more meteors needed to be spawned near it, and we would spawn some if we did need to spawn more meteors.

Opting to use C# over Godot's GDScript had its ups and downs. For one, I was much more familiar with its paradigms, so it was easier to hit the ground running. However, Godot 4's lack of HTML5 export support meant that I was ultimately unable to create an in-web playable export of my program.

1 - Talk about the unique aspects such as how it relates to 'space' in the gj prompt
2 - Write for a technical audience, try to use broadly understood terms
