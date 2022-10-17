+++
title = "you SHOULD store session tokens in local storage"
date = 2022-10-14
draft = true

[taxonomies]
tags = ["security", "web"]
+++

> **TLDR**: We should store session tokens in local storage, not cookies. Here's why.

## what are session tokens?

Session tokens are used to authenticate users on a website. They're usually stored in cookies, and they're sent to the server on every request. The server can then use the session token to identify the user.

## why should we store them in local storage?

Storing session tokens in cookies has a few issues:

- Cookies increase the size of every request, which can have a negative impact on performance.
- Cookies are always sent to the server, even if you don't need them. This means exposing the session token to the server unnecessarily.
- Cookies can lead to CSRF attacks.

Storing session tokens in local storage solves all of these issues. Local storage is only sent to the server if you explicitly request it. This means that you can send the session token to the server only when you need it and you can't accidentally expose the session token to the server. Especially when using a CDN, this can prevent your session tokens from being exposed to the server. Things like CSRF attacks are also much harder to pull off, since you can't access local storage from a different domain.

## what about XSS attacks?

XSS attacks are a problem when you store session tokens in local storage. This is because local storage is accessible from JavaScript which means that an attacker can read the session token from local storage, and use it to impersonate the user. I would argue that this is a much smaller problem than the issues that come with storing session tokens in cookies. Once a XSS attack has happened, you can't do much to prevent an attacker from interacting with your API anyways.

If you're worried about XSS attacks, you should be using a framework like React or Vue, which prevents XSS attacks by default. If you're not using a framework, you should be using a library like DOMPurify to sanitize user input. This will prevent XSS attacks from happening in the first place. Additionally, you should be using a strong Content Security Policy to prevent XSS attacks from happening in the first place.
