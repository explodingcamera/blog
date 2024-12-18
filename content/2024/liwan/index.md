---
title: Lightweight Web Analytics with Liwan
description: "A look at my new project Liwan, a lightweight, privacy-focused web analytics tool."
date: 2024-12-18
---

> This post explores some of this project's background and technical aspects. If you're more so interested in the Liwan itself, you can check out the [demo instance](https://demo.liwan.dev/p/liwan.dev) and the docs on [liwan.dev](https://liwan.dev). The source code available on [GitHub](https://github.com/explodingcamera/liwan) under the AGPL-3.0 license.

This summer, I started working on a small tool for collecting various metrics on my websites and this blog, partly because it helps me see what's popular and partly just as a feel-good vanity thing.

{{ figure(caption = "The Liwan Dashboard for one of my websites.", position="center", src="./dashboard.jpg") }}

Over the last ~5 years, I've tried out probably ten different analytics platforms after bailing on Google Analytics due to privacy concerns, but nothing ticked all the boxes for me:

**I wanted strong privacy guarantees**. First of all, because it should be the default across all websites, but also because of how (rightfully) painful GDPR compliance can be when you collect too much data. I don't need to know your IP address or your browsing habits. There's no real reason for this data to leave my server; even when anonymized, there's always a risk of being misused.

**It should be set and forget**. I love that self-contained, statically linked binaries are making a comeback due to being the default with "newer" languages such as Go and Rust, which is fantastic for self-hosted software. Around the time I started working on this project, DuckDB 1.0 was also announced and later released, which seemed like a perfect tool for this. I value a simple setup process more than the ability to hyper-scale prematurely and endlessly customize everything.

**Truly Open Source**: One of my favorite tools I tried was [fathom analytics](https://usefathom.com/), which sadly (and understandably) switched to a completely closed source, cloud model. I want to 'own' the data, even if the user's data is eventually anonymized; it's hard to trust what's happening on a server I don't control. I really like [plausible](https://plausible.io/)'s open core model; I just want something that is a bit more lightweight.

**Multi Website**: As I mentioned, I want to aggregate data from 10+ different sites, so multi-website support also had to be great. One platform I experimented with early on was (GoatCounter)[https://www.goatcounter.com/]. While the UI is bare bones, it's also very lightweight and available with an embedded database. Sadly, it does not offer multi-website support and is generally not as feature-rich as I would like.

Setting out with those and some more goals in mind, I started building [Liwan.dev](https://liwan.dev), which, as often happens, had a lot of feature creep but is now finally in a state where it could also be useful for a lot of small to medium sites.

# Simple Software

I love abstractions. Not in the OOP or the JavaScript way but in how a robust, tested, and, most importantly, focused library or app feels to use. This is one of the reasons I am so drawn to software engineering in the first place: the ability always to go one level deeper and understand how things work.

You can overplay this - see the NPM ecosystem (this is honestly a bit of a strawman, [left-pad](https://en.wikipedia.org/wiki/Npm_left-pad_incident) was not that bad), but it is a nice contrast from the attention-seeking everything apps and magic frameworks. There is a hard-to-find balance here - I won't argue in favor of building on hundreds of layers of magic and abstractions even when ignoring the supply chain concerns - but I really enjoy the current balance in the Rust ecosystem. The language itself doesn't matter, though; I mainly use it because I enjoy working with it.

Liwan is built on hundreds of open-source libraries; you can see the list for yourself on the [attributions page](https://demo.liwan.dev/attributions) shipped with every copy of Liwan. However, this doesn't mean it's not lightweight. Even when using the 'modern' web stack, resource-heavy and slow websites don't have to follow automatically. The dashboard is built using [Astro](https://astro.build/), a web framework built around reducing unnecessary overhead.

A big part of the complexity of similar projects often comes from overly customizable Graph libraries. To reduce the amount of code that needs to be sent to the user, I build custom Graph and Map components directly on [d3](https://d3js.org/), keeping the entire JavaScript bundle shipped to users below 250kb. This even includes a large number of different icons and the data for the world map, which uses an optimized [topojson](https://github.com/topojson/topojson) file I created with data from the [natural earth project](https://www.naturalearthdata.com/).

{{ figure(caption = "Liwan's PageSpeed Insights.", position="center", src="./pagespeed.jpg") }}

On the backend side, the main contributors to the amount of code are Rustls and the embedded databases, DuckDB for events, and SQLite for the user data and authentication. The dashboard is transformed into static HTML, CSS, and JS and bundled with the rest of the code, something I've recently started doing with some of my other projects as well. Producing a single, universal artifact simplifies the entire build process and packaging containers greatly (Something that could be pushed even further using [Actually Portable Executable](https://justine.lol/ape.html)). To maximize compatibility across different Linux distro (you might have had issues with glibc on alpine containers before if you've worked with docker imaged), Liwan is also statically linked with [musl libc](https://musl.libc.org/) and uses [rustls](https://github.com/rustls/rustls) instead of linking against OpenSSL.

Simple software doesn't stop here, however. Liwan is also built to require only a minimal amount of configuration and settings from the user before it is ready to process events. You execute the binary, and Liwan is ready. Everything will be placed in the correct place according to XDG Base Directory Specification, and you can start sending events to it right away.

Early on, I also decided to add an onboarding page for users to create their initial user account. This page can only be accessed from a URL printed to the console on the first startup, protecting you from accidentally exposing this screen to the public internet and removing the need for default passwords.

# Open Source

As a small side note on the open-source part, I've thought a lot about how I wanted to license this project. I ended up going with the AGPL-3.0, which I'm not 100% happy with but is the best compromise for now. My one big gripe with the AGPL-3.0 is the mostly undefined virality boundaries, which probably apply less in the EU (to remedy this a bit, the tracker script is also available under the MIT license). In other projects, I usually use the MIT + Apache 2.0 dual license that is so common in the Rust ecosystem. Still, I want flexibility to allow me to monetize Liwan more easily in the future. To have the possibility to relicense it later and not need to set up a CLA, all contributions also need to be provided under the MIT license as well, something I haven't seen in many other projects and seems like a good compromise for both sides.

The main alternative I considered was the EUPL. I've only seen it used on very few projects, but it's a bit more permissive than the AGPL. The author of GoatCounter has some interesting thoughts on why he chose it on his [blog](https://www.arp242.net/license.html). I don't like the license's wording: It allows relicensing the work under entirely different terms, and it just feels like a mess to me. It tries to be an "Interoperable Copyleft" license and explicitly states it's compatible with the GPL 2 and 3, but the GPL itself is _probably_ incompatible with the EUPL ([see gnu.org](https://www.gnu.org/licenses/license-list.html#EUPL-1.1)), _except_ if you do a weird two-step relicensing dance. But I've [read in forums](https://interoperable-europe.ec.europa.eu/collection/eupl/discussion/how-does-fsf-considers-eupl) that it still keeps the terms of the EUPL, so relicensing doesn't actually weaken it? I don't know, it's just a mess.

# Try it out!

Liwan is not perfect. There are many things I could optimize more, starting with huge datasets - analyzing millions of events takes a lot of resources - but it fits my personal use case (nearly) perfectly and probably yours, too.

Just grab the latest binary to try it out:

```bash
# Download the latest release
curl -JLO 'https://github.com/explodingcamera/liwan/releases/latest/download/liwan-x86_64-unknown-linux-musl.tar.gz'

# Ensure the ~/.local/bin directory exists (You might want to add it to your PATH)
mkdir -p ~/.local/bin

# Extract the binary
tar -xzf liwan-x86_64-unknown-linux-musl.tar.gz -C ~/.local/bin liwan

# Make the binary executable
chmod +x ~/.local/bin/liwan

# Run the binary
liwan --help
```
