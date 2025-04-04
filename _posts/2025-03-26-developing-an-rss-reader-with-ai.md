---
layout: post
title: "Developing a feed reader with an AI assistant"
date: "2025-03-26 22:58:00 -0300"
excerpt: How I defeated inertia and (re)wrote a feed reader using GitHub Copilot.
---

This is the story of why I use feed readers, how I used LLMs to rewrite my personal reader web application, and what the experience was like.

[Web feeds](https://en.wikipedia.org/wiki/Web_feed) are a precursor to what we today know as social media. They are a way to subscribe to sources (usually websites) and receive their updates without having to individually visit every website. The idea never really became widespread, because social networks quickly borrowed the concept and packaged it in a more easy-to-use format (the social feed). And then these social networks mostly became horrible places. But that's beyond the point. Web feeds still exist today, and two formats are dominant: RSS and Atom.

I started using feed readers back in 2004 or 2005, when blogs were all the rage. I had read a blog post explaining how feeds work and got interested. I started with [Liferea](https://en.wikipedia.org/wiki/Liferea), then moved on to Google Reader, until it was discontinued in 2013, because of a social network (Google+)...

I then moved on to Feedly, which was nice, but it just didn't have that same feeling as Google Reader had. Sometime in 2020, they started showing sponsored ads in the feed list (which is completely fair, they have to make money), so I decided to move away.

I am cheap and didn't want to pay for a feed reader, so I was bound to be disappointed by ads eventually. I also wanted a (web?) app that could be used on both my desktop and on my phone. I never had too many feeds: usually the number floated around 30.

That's when I started toying with the idea of writing my own feed reader, in a way that could be self-hosted in the free tiers offered by PaaS providers, namely Heroku. I went with Python + Postgres, and in a couple of days I had a functional feed reader. I didn't devote much effort, did not write tests for it, and built just a minimal API and a single HTML file with some JavaScript as the web app. I kept on using that feed reader for years. When Heroku ended their free tier, I moved it to Azure App Services, using SQLite.

But then Mozilla deprecated the [bleach](https://github.com/mozilla/bleach) HTML sanitization library I was relying on. And the performance of even very simple SQLite usage on the free tier of Azure became terrible -- think 10s+ to mark all feeds as read. For some reason, the Python container image used in Azure was also consuming a ton of resources, sometimes making the app exceed the free quota. So I had to breathe some life back into my feed reader.

Back in 2020 I had only been using Go for a year, so I went with Python when writing the original reader. By 2025, I had come to deeply admire Go and the culture around it. So I wanted to rewrite it in Go, but I was demotivated by having to repeat all of the effort, select and learn new dependencies, etc. That's when the AI hype finally struck me. I had been meaning to use [GitHub Copilot](https://github.com/features/copilot), and decided this might be a good use case for it.

I started with the most annoying part: the HTML sanitizer. I wanted something very simple to maintain, ideally something that would not be abandoned later. I tasked Copilot with writing one, inspired in the way I was invoking bleach in the Python-based feed reader. And in less than 2 hours of tweaking and thinking about the problem, I had a good-enough-for-me HTML sanitizer with decent test coverage, using nothing more than Go's `x/net` module. You can see it [here](https://github.com/alnvdl/varys/blob/main/internal/fetch/sanitize.go).

You may look at that code and think: "It took you 2 hours to write 100 lines of code with AI???", to which I would reply: "No, it took me 2 hours to understand the problem and iteratively come up with the most minimalistic solution I could with AI." :-) It was a very enjoyable process, akin to pairing with a fast and smart coworker.

> I would have written a shorter letter, but I did not have the time.
>
> -- [*Blaise Pascal*](https://en.wikiquote.org/wiki/Blaise_Pascal)

Then I ported the RSS/Atom XML parsing logic. I also took the chance to add an HTML scraper for sites that don't provide feeds, and another one that builds a feed based on any URL hosting an image that can change with time. You can find all of that [here](https://github.com/alnvdl/varys/tree/main/internal/fetch).

To further reduce dependencies, I decided to periodically persist the data to a file, and not a database, which you can see [here](https://github.com/alnvdl/varys/tree/main/internal/list/mem). With more help from Copilot, I ported the API from Python into Go ([here](https://github.com/alnvdl/varys/tree/main/internal/web)), relying on [Go's recent routing enhancements](https://go.dev/blog/routing-enhancements). Then I copied the HTML + JavaScript file from the Python feed reader, and I had a perfect replacement for it!

Along the way, I used Copilot to write tests and documentation. Now I have a better feed reader than I had before, (practically) without external dependencies, much faster (the in-memory data storage really helps) and much more maintainable in the long term.

![The Varys feed reader showing the feed for this blog.](/assets/img/varys_screenshot.png)
<p class="caption">
The <a href="https://github.com/alnvdl/varys">Varys feed reader</a> showing the feed for this blog.
</p>

You can find the code and documentation here: [https://github.com/alnvdl/varys](https://github.com/alnvdl/varys).

## On my experience using AI for coding

By far, the greatest boost provided by Copilot was breaking the inertia involved in starting the project. Once you have *something*, *anything*, it's much easier to iterate and get things going.

Copilot (and other LLM-based tools in general) are not perfect by any means, and they still make mistakes all the time. Very much like myself :-) So there is the constant need for verifying what it has done and refine it, just like a good engineer will always second-guess themselves. But the tooling to assist in that iterative process is already great, and it's only improving.

I wouldn't describe what I did as [vibe coding](https://en.wikipedia.org/wiki/Vibe_coding). Throughout the process, I was engaged, making decisions and manually coding parts I considered critical to my self-defined requirements: no dependencies, well-documented, well-tested, minimalistic code. But it wasn't regular coding either: I was much more inclined to ask Copilot to try experimental half-baked ideas, knowing it was cheap to throw them away.

Writing tests in particular was a great experience. I really enjoyed the process of thinking about the tests, telling Copilot to write them (listing all cases), and then going for a quick break. More often than not, I would come back to almost perfect results, requiring very little tweaking. Refactoring tests later was equally a breeze.

I can imagine people seeing this kind of tooling and thinking it replaces software engineers. But when paying for a software engineer, one is actually paying for building knowledge, not just for coding. The code is just a way to encode that knowledge. And now with LLMs, we have a new, much improved way to get computers to encode knowledge.

It may very well be the case that soon we will have AIs capable of building knowledge iteratively and collaboratively. That will be the day when software engineers like myself (and other knowledge workers) will have to go fulfill their long-imagined dreams of owning a farm or starting a restaurant :-)
