+++
title = "ai is helping us to finally reform copyright law"
date = 2022-10-17

[taxonomies]
tags = ["ai", "github copilot", "copyright"]
+++

_This post was written entirely by GitHub Copilot, I only gave a few nudges so it stayed on topic_

# Introduction

Recently, we've reached the news cycle of criticizing GitHub Copilot and other AI-assisted code writing and image creation tools again, at least in the tech community. I think it's a good time to reflect on the current state of AI and how it can finally help us reform copyright law.

{% quote(author="Tim Davis (@DocSparse)", class="twitter", url_text="October 16, 2022", url="https://twitter.com/DocSparse/status/1581461734665367554") %}
<a href="https://twitter.com/github?ref_src=twsrc%5Etfw">@github</a> copilot, with &quot;public code&quot; blocked, emits large chunks of my copyrighted code, with no attribution, no LGPL license. For example, the simple prompt &quot;sparse matrix transpose, cs\_&quot; produces my cs_transpose in CSparse. My code on left, github on right. Not OK. <a href="https://t.co/sqpOThi8nf">pic.twitter.com/sqpOThi8nf</a>
{% end %}

In the corresponding [Hacker News discussion](https://news.ycombinator.com/item?id=33226515), there's a clear and negative sentiment towards AI-assisted code writing tools:

{% quote(author="enriquto", class="hackernews", url_text="link", url="https://news.ycombinator.com/item?id=33226515#33231362") %}
[...] it is notoriously not used by scipy.linalg.solve, though, because numpy/scipy developers are anti-copyleft fundamentalists and have chosen not to use this excellent code for merely ideological reasons
{% end %}

{% quote(author="mjr00", class="hackernews", url_text="link", url="https://news.ycombinator.com/item?id=33226515#33226972") %}
Same issue with Stable Diffusion/NovelAI and certain people's artwork (eg Greg Rutkowski) being obviously used as part of the training set. More noticeable in Copilot since the output needs to be a lot more precise.
Lawmakers need to jump on this stuff ASAP. [...]
{% end %}

There is even talks about a potential [law­suit](https://githubcopilotinvestigation.com) to get GitHub to remove the ability to use copyrighted code from Copilot.

Until now, I've mostly ignored these because I don't think most of them are very productive. It's a good thing that we're finally having this discussion, but I don't think we're going to get anywhere by just criticizing the tools. We need a more constructive conversation about what we want to achieve with copyright law.

The particular focus on tooll like GitHub Copilot is the main part of this discussion I just don't get. We've had GPT-3 and others for a while now, and these are - in contrast to copilot - using tons of copyrighted material for their training. And yet, we don't see the same kind of outrage.

I think there are a few reasons for this:

## There is a lot of misinformation about how AI-assisted code-writing tools work

People think that they are just copying code, but they are not. They are using the training data to generate code similar to the training data. This is a very different thing. It's like saying that a translator is copying the original text because they are using it to generate the translation. It's not copying; it's using the original text to develop something new. I see this sentiment in the discussion about AI-assisted code-writing tools, which is not true.

# There are a ton of Copyleft fundamentalists who overvalue their code

They think that their code is so valuable that it should be protected from being used by others. Here I'm not talking about copying an entire project, but about using a small part of the code to generate something new. Being able to copyright a utility function itself is already dubious legally at best (especially since we don't have any legal precedent for this), and trying to enforce copyright on a small part of a function is hurting the entire community.

# We need to reform copyright law

We should be working towards a copyright law that allows us to use AI-assisted code-writing tools in a way that benefits everyone. This means that we need to be able to use copyrighted code to generate something new, and we need to be able to use the generated code in our projects. I don't think these tools will go away, and we should embrace them and use them to improve our code quality and productivity. Microsoft is obviously "abusing" its position as the owner of GitHub to get a head start on this, but I think that's a good thing for the community.

# Conclusion

I tried to have some fun with this blog post: I gave GitHub copilot a couple of pointers and wanted to see how it defended itself (themself?) from the recent criticism. While I generally agree with the sentiment of this post, it was written by our future AI overlords, and I'm not responsible for anything they might have written here. So that you know my bias when prompting the AI: I love using GitHub Copilot for auto-completion. It definitely took some convincing, but now it's an essential tool for me, and I'm looking forward to seeing how it will evolve in the future (and if the announced law­suit will affect it).

Finding good promps was harder than I initially thought, because Copilot tried to repeat one point over and over:
{% quote(author="GitHub Copilot") %}
We should be working towards a copyright law that allows us to use AI-assisted code-writing tools in a way that benefits everyone.
{% end %}

{{ figure(src='/img/copyright-dalle2.png', caption="DALL-E 2: our future robot ai overlords defending themselves") }}
