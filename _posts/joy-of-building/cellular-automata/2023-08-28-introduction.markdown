---
layout: post
title: "Cellular Automata: introduction"
categories: joy-of-building cellular-automata
date: 2023-08-28 21:00:00 +0200
tags:
    - cellular-automata
    - wolfram
    - conway
    - gaming
    - procedural-generation
---

# Cellular Automata: introduction

## Curiosity Goals

Every year I like to concoct three light-hearted goals, not meant to be serious and just to throw me in the direction of something new. The goals specify a desired outcome as opposed to a process, to prevent getting locked down on something I may grow to despise during the year. A simple example is to read 6 books, without mention which books, what genre, just books. Another would be to invite and cook for X amount of people. The goals can morph, but in order for them to still mean something I make sure to allocate some time at the expense of day-to-day recreational activities.

This year I had the intention of making a video game. With no prior experience in game development, it presented itself as not just an unwandered road, but an unlit one. Having a goal like this is fun, you can modify or explore specific parts that grab your attention, as you will see in my case. Truth be told, the goal is most likely too ambitious, but I have already started wandering in some areas, lighting up the darkness. That still counts as progress.

Why a game? There may a thousand answers to this question. Instead of covering them all I would like to cover some high level ideas of mine. First, developing a game seems like not just a programming experience, but a way to express oneself in an art form. You can pour as much personality into it as you want to, or you can approach it as a strictly technical challenge. A lot of professional development work can be tedious and boring, even numbing one's creativity. it is still very challenging yes, but truth be told, I always remember why I appreciate IT and software development the most when working on something after hours.

See, with a bit of grit you can grind through learning a frontend language, you can now design and implement those potential 1 million dollar idea web apps. Bite through some backend guides, and now you can publish those ideas for everyone to see, facilitating some aspect of a person's life. With AI, you can think of wicked ideas and combined with frontend and backend, serve it up for enthusiasts around the globe. It's not for nothing that these blog posts are in a category called "joy-of-building".

Then there's gaming itself. Though gaming is not that prominent (anymore) in my life, games such as [Valheim](https://www.valheimgame.com/), [Rimworld](https://rimworldgame.com/) and [Minecraft](https://www.minecraft.net/en-us) enabled me to immerse myself and create a little world. What these games have in common is a degree of procedural generation, specifically the terrain you navigate. There is something that tickles my fascination here: giving character to locations. And there is a sense that more is possible. A coast of cliffs eternally showered in torrential rains; a savannah scorched by the blistering sun, its sparse vegetation offering refuge to lifeforms. A sunny vale with temperate weather year-round. Perhaps now you imagine real places you have been to, and you recall the exact sentiments they evoke. That in short is what I envision. A location with everpresent character, determined by many many variables, but each location is not in isolation of one another. No, they are interwoven, separated by distance and either gradual or sudden shift in character. If you conjure up an elaborate world, does the world itself not become an entity, thus immersing the player.

I like the idea that this character itself might be procedurally generated. Why limit yourself to terrain, when weather, life, wind, resources and much more can be part of the picture as well? Our quest is going to be one of looking through mathematics and science to try and bring about this vision. At this point, I do not know of all the things that will cross our path yet, but we do have a first direction: [cellular automata](https://en.wikipedia.org/wiki/Cellular_automaton).

## Checkpoints

This article will evolve over time, with the intent of adding checkpoints of major search directions related to the topic of cellular automata.

### Summer 2023: A New Kind of Science

Most people will have heard of [Wolfram](https://www.wolfram.com/), a website offering many mathematics guides and definitions. The person behind is [Stephen Wolfram](https://nl.wikipedia.org/wiki/Stephen_Wolfram). In the 80s he started delving into cellular automata and became enamored about how such simple dynamics can give rise to complex phenomena, an early discovery of an [emergent system](https://en.wikipedia.org/wiki/Emergence). By now, most people will have crossed paths with [Conway's Game of Life](https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life).

In my university days I had already had some run-ins with cellular automata. In one course, ex-students would be invited to give lectures on issues they were working on in the industry. One in particular, [Transport & Mobility Leuven](https://www.tmleuven.be/nl/person/sven-maerivoet), uses them to simulate traffic. Such experiments allow to model dynamics given traffic conditions to test infrastructure proposals. The whole idea of simple dynamics being able to generate such complex macro behaviour seemed unreal. There are far more fields where cellular automata are used, and we advise the user to explore these.

I wanted to know more about the theory behind it, and luck has it that Wolfram himself published his work for free [online](https://www.wolframscience.com/nks/). These documents will form the base to branch out of, and in subsequent blog posts we intend to not only learn, but create interactive demos for others to explore.

So without further ado, let's kick off an iterative process of reading, followed by implementation and experimentation. We will add links here over time to document out progress.

| Class | Vault | Article |
|-------|-------|---------|
| simple | [Link](https://vault.dev.brainfartlab.com/cellular-automata/nks/simple/) | [Link]({% post_url joy-of-building/cellular-automata/2023-09-07-simple %}) |
| mobile | [Link](https://vault.dev.brainfartlab.com/cellular-automata/nks/mobile/) | [Link]({% post_url joy-of-building/cellular-automata/2023-09-12-mobile %})|
