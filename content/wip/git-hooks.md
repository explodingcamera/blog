+++
title = "git pre-commit hooks are a bad idea"
date = 2022-10-14
draft = true

[taxonomies]
tags = ["git", "workflow"]
+++

> **TLDR**: We sould stop using git pre-commit hooks for code formatting, linting and testing. Instead, we should only later use CI/CD to enforce these rules.

Over the last few years, I've heard a lot of different opinions on the use of git pre-commit hooks. Some people swear by them, others think they're a bad idea. I've been using them for a while now, and I've come to the conclusion that they're a bad idea. Here's why.

## what are git pre-commit hooks?

Git pre-commit hooks are scripts that run before you commit your changes. They can be used to run tests, lint your code, or even format your code. They're a way to enforce certain rules on your codebase. For example, you can use a pre-commit hook to run your tests before you commit your changes. If the tests fail, the commit will be aborted. This way, you can be sure that your code is always in a working state.

## why are they a bad idea?

My main issue with git pre-commit hooks is their impact on development velocity. Especially when you're working on a team, you want to be able to commit your changes as often as possible. If you have to wait for your tests to run before you can commit, you'll end up committing less often.

## what should you use instead?

They're also often a symtom of a larger problem. If you're using pre-commit hooks to enforce testing, formatting and linting, you're wasting a lot of time and energy on things that should be handled by you editor. If you're using a modern editor, it should be able to format your code on save, and it should be able to run your tests and linters on the fly. This ensures that you can get instant feedback on your code.

Having a CI/CD pipeline that runs your tests and linters is also a good idea. This way, you can be sure that your code is always in a working state, even if you forget to run your tests before you commit.
