---
layout: post
title: "QuizCraft: the reason"
categories: joy-of-building quiz-craft
date: 2023-07-17 14:00:00 +0200
tags:
    - elm
    - auth0
---

# Quiz Craft: the reason

Every year around the holidays our family has a tendency (slowly becoming a tradition) to buy some packs of quiz cards. Some of us get competitive, others just like to nerd out. We make our way through the card packs relatively fast. With the obsession around large language models I figured why not have ChatGPT or some other LLM generate question given a number of keywords? The questions will never run out then. We'll wrap it up in a web application for easy use. This is the practical reason.

Then there's the technical reasons. I come from a backend background, mainly using cloud computing platforms. My personal AWS setup uses Control Tower for multi-account organization to mimic use cases in my freelancing career. So I figured, why not overengineer by design? I'll have separate accounts for each environment, backed by CI/CD. If my goal was to just get a backend API out the door, anyone would stick a to a simple single account setup. But I have seen that plenty times. We will tackle the backend in a more "corporatey" style. Our attention to learning and creating is 50-50.

A second aspect will be the frontend. For too long have I thought "If only I knew frontend, who knows what wonders I could make". Well, for this project we will get out of our comfort zone and prove to ourselves that we truly lack innovative ideas. But at least we will know frontend programming.

The third and last aspects will be machine learning. With the emergence of large language models we already see mentions of "prompt engineering"; how to prompt the LLM to get the desired result. A team centered around [Andrew Ng](https://www.andrewng.org/) has laid the groundwork with a package called langchain that we can take as a starting point. It's not hardcore mathematics, but I reckon it will still be plenty challenging.

Each of the three aspect will get its own dedicated blog post.
