---
title: Lightweight Web Analytics with Liwan
date: 2024-12-06
draft: true
---

> This post explores a bit of the background and technical aspects of this project. If you're more so interested in the Liwan itself, you can check out the [demo instance](https://demo.liwan.dev/p/liwan.dev) and [liwan.dev](https://liwan.dev).

This summer, I started working on a small tool for collecting various metrics on my websites and this blog, partly because it helps me see what's popular and partly just as a feel-good vanity thing.

Over the last ~5 years, I've tried out probably ten different analytics platforms after bailing on Google Analytics due to privacy concerns, but nothing ticked all the boxes for me.

**I wanted strong privacy guarantees**. First of all, because it should be the default across all websites, but also because of how (rightfully) painful GDPR compliance can be when you collect too much data. I don't need to know your IP address or your browsing habits.

**It should be set and forget**. I love that self-contained, statically linked binaries are making a comeback due to being the default with "newer" languages such as Go and Rust, which is fantastic for self-hosted software. Around the time I started working on this project, DuckDB 1.0 was also announced and later released, which seemed like a perfect tool for this.

**Truly Open Source**: One of my favorite tools I tried was [fathom analytics](https://usefathom.com/), which sadly (and understandably) switched to a completely closed source, cloud model. I want to 'own' the data, even if the user's data is eventually anonymized; it's hard to trust what's happening on a server I don't control. I really like [plausible](https://plausible.io/)'s open core model; I just want something that is a bit more lightweight.

**Multi Website**: As I mentioned, I want to aggregate data from 10+ different sites, so multi-website support also had to be great. One platform I experimented with early on was (GoatCounter)[https://www.goatcounter.com/]. While the UI is bare bones, it's also very lightweight and available with an embedded database. Sadly, it does not offer multi-website support and is generally not that flexible.

Setting out with those and some more goals in mind, I started building [Liwan.dev](https://liwan.dev), which, as often happens, had a lot of feature creep but is now finally in a state where it could also be useful for a lot of small to medium sites.

# Simple Software

I love abstractions. Not in the OOP or the JavaScript way but in how a robust, tested, and, most importantly, focused library or app feels to use. You can overplay this - see the NPM ecosystem (this is honestly a bit of a strawman, [left-pad](https://en.wikipedia.org/wiki/Npm_left-pad_incident) was not that bad), but it is a nice contrast from the attention-seeking everything apps and magic frameworks. There is a hard-to-find balance here - I won't argue in favor of building on hundreds of layers of magic and abstractions even when ignoring the supply chain concerns - but I really enjoy the current balance in the Rust ecosystem.

Liwan itself is built on hundreds of open-source libraries; you can see the list for yourself on the [attributions page](https://demo.liwan.dev/attributions) shipped with every copy of Liwan. However, this doesn't mean it's not lightweight. Even when using the 'modern' web stack, resource-heavy and slow websites don't have to automatically follow. The Dashboard is built using [Astro](https://astro.build/), a web framework built around reducing unnecessary overhead.

A big part of the complexity of similar projects often comes from overly customizable Graph libraries. To reduce the amount of code that needs to be sent to the user, I build custom Graph and Map components directly on [d3](https://d3js.org/), keeping the entire JavaScript bundle shipped to users below 250kb. This even includes a large number of different icons and the data for the world map, which uses an optimized [topojson](https://github.com/topojson/topojson) file I created with data from the [natural earth project](https://www.naturalearthdata.com/).

On the backend side, the main contributors to the amount of code are Rustls and the embedded databases, DuckDB for events, and SQLite for the user data and authentication. However, after compressing the binaries, they only come in at about 14MB. This includes the entire dashboard website itself, which is bundled with the rest of the code, something I've recently started doing with some of my other projects as well. Producing a single, universal artifact simplifies the entire build process and packaging containers greatly (Something that could be pushed even further using [Actually Portable Executable](https://justine.lol/ape.html)). To maximize compatibility across different Linux distro (you might have had issues with glibc on alpine containers before if you've worked with docker imaged), Liwan is also statically linked with [musl libc](https://musl.libc.org/).

Simple software doesn't stop here, however. Liwan is also built to require only a minimal amount of configuration and settings from the user before it is ready to process events. You execute the binary, and Liwan is ready. Everything will be placed in the correct place according to XDG Base Directory Specification, with no subtle differences between different operating systems.

Early on, I also decided to add an onboarding page for users to create their initial user account. This page can only be accessed from a URL printed to the console on the first startup, protecting you from accidentally exposing this screen to the public internet and removing the need for default passwords.

# Try it out!

Liwan is not perfect. There are many things I could optimize more, starting with huge datasets - analyzing millions of events takes a lot of resources - but it fits my personal use case (nearly) perfectly and probably yours, too.
