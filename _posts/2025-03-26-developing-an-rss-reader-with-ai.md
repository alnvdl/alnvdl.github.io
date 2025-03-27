---
layout: post
title: "Developing an RSS reader with AI"
date: "2025-03-26 22:58:00 -0300"
excerpt: How I defeated inertia and (re)wrote an RSS reader using GitHub Copilot.
---

I started using RSS readers back in 2004 or 2005, when blogs were all the rage. I had read a blog post explaining how feeds work and got interested. I started with [Liferea](https://en.wikipedia.org/wiki/Liferea). After a while, I moved to Google Reader, and kept on using it until 2013, when it was discontinued, because all good things must come to an end...

I then moved on to Feedly, which was nice, but it just didn't have that same feeling that Google Reader had. Sometime in 2020, they started showing sponsored ads in the feed list (which is completely fair, they have to make money), so I decided to move away.

I just didn't know which feed reader to start using: I am cheap and didn't want to pay for it, so I was bound to be disappointed by ads eventually. I also wanted a (web?) app that could be used on both my desktop and on my phone. I never had too many feeds, usually the number floated around 30 feeds.

That's when I started toying with the idea of writing my own RSS reader, in a way that could be self-hosted in the free tiers offered by PaaS providers, namely Heroku. I went with Python + SQLite, and in a couple of days I had a functional feed reader. I didn't devote much effort, did not write tests for it, and built just a minimal API and a single HTML file with some JavaScript (SPA style). I kept on using that small reader I developed from early 2021 for years. When Heroku ended their free tier, I moved it to Azure App Services.

But then Mozilla deprecated the [bleach](https://github.com/mozilla/bleach) HTML sanitization library I was relying on. And the performance of even very simple SQLite usage on the free tier of Azure became terrible (think 10s+ to mark all feeds as read). For some reason, the Python container image used in Azure was also consuming a ton of resources, sometimes making the app exceed the quota.

I had to do something to improve the situation, I just didn't know what to do.

Back in 2020 I had only been using Go for a year. By 2025, I've come to deeply admire the language and the culture around it. What if I rewrote it in Go? But I was demotivated by having to select dependencies again, and then rewrite the whole thing for what should just be a hobby project. That's when the AI hype finally struck me. I had been meaning to use GitHub Copilot, and decided this might be a good use case for it.

I started with the most annoying part: the HTML sanitizer. I wanted something stupid simple to maintain, ideally something that would not be abandoned later. I tasked Copilot with writing one, inspired in the way I was invoking bleach in the Python-based feed reader. And in less than 2 hours of tweaking and thinking about the problem, I had a good-enough-for-me HTML sanitizer with decent test coverage, using nothing more than Go's `x/net` module. You can see it [here](https://github.com/alnvdl/varys/blob/main/internal/fetch/sanitize.go).

You may look at that code and think: "It took you 2 hours to write 100 lines of code with AI???", to which I would reply: "No, it took 2 hours for me to understand the problem space and iteratively come up with the most minimalistic solution I could with AI." :-) It was a very enjoyable process, akin to pairing with a fast and smart coworker.

> I would have written a shorter letter, but I did not have the time.
>
> -- [*Blaise Pascal*](https://en.wikiquote.org/wiki/Blaise_Pascal)

Then I ported the RSS/Atom XML parsing logic. I also took the chance to add an HTML scraper for sites that don't provide feeds, and another one that builds a feed based on any URL hosting an image that can change with time. You can find all of that [here](https://github.com/alnvdl/varys/tree/main/internal/fetch).

In the end, I decided to not have a database, and just write to keep the entire feed list and contents in memory, dumping it to a file periodically, which you can see [here](https://github.com/alnvdl/varys/tree/main/internal/list/mem). With help from Copilot help, I ported the API from Python into Go ([here](https://github.com/alnvdl/varys/tree/main/internal/web)). Then I copied the HTML + JavaScript file from the Python application, and I had a perfect replacement for my old Python feed reader!

Along the way, I used Copilot to write tests and documentation. And now I have a better feed reader than I had, (practically) without external dependencies, infinitely faster (the in-memory thing really helps) and much more maintainable in the long term.

![The Varys feed reader showing the feed for this blog.](/assets/img/varys_screenshot.png)
<p class="caption">
The <a href="https://github.com/alnvdl/varys">Varys feed reader</a> showing the feed for this blog.
</p>

You can find the code and documentation here: [https://github.com/alnvdl/varys](https://github.com/alnvdl/varys).

## On my experience with using AI for coding

By far, the greatest boost provided by Copilot was breaking the inertia involved in starting a project. Once you have *something*, *anything*, it's much easier to iterate and get the project going.

Copilot (and AI tools in general) are not perfect by any means, and they still make mistakes all the time. Very much like myself :-) So there is the constant need for verifying what it has done and refine it, just like a good engineer will always second-guess themselves. But the tooling to assist in that iterative process is already great, and it's only improving.

Writing tests in particular is a great experience. I really enjoyed the process of thinking about the tests, telling Copilot to write them (listing all cases), and then going for a quick break. More often than not, I would come back to almost perfect results, requiring very little tweaking. Refactoring tests later was equally a breeze.

I can imagine people seeing this kind of tooling and thinking it replaces software engineers. But when paying for a software engineer, one is actually paying for building knowledge, not just for coding. The code is just a way to encode that knowledge. And now with LLMs, we have a new, much improved way to get computers to encode knowledge.

It may very well be the case that soon we will have AIs capable of building knowledge iteratively and collaboratively. That will be the day when software engineers like myself (and other knowledge workers) will have to go fulfill their long-imagined dreams of owning a farm or starting a restaurant :-)
