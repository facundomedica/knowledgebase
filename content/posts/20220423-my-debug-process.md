---
title: "My debug process"
author: "Facundo Medica"
type: "post"
date: 2022-04-23T10:10:02-03:00
draft: false
subtitle: "It's not special at all, just how I do it :)"
image: ""
tags: ["go"]
---

This is my process to figure out what's wrong with pretty much anything (not that I can figure out everything, but it's my go-to process to everything). It's language agnostic and what I suppose you can call "fast paced".

I think it's specially useful when debugging code bases I don't know, given that I can't assume anything, just follow the code and see where it leads me to. But as always, YMMV.

### 1. Search for keywords, find out what's the dependency that holds your attributes of interest.

Main tools:

- `cmd/ctrl + click` to follow where things come from
- Search bars. I use my IDE's, Github and Google to find the origin of anything

The process is simple, if you don't know where something is defined you just search it everywhere you can until you see a meaningful result. If you already know where something is coming from, then start going around doing cmd+click at stuff.

### 2. Log log log

Use `fmt` or `log` packages (or any other log package available in the code you are debugging) to print:

- Objects and attributes that could be remotely related to the issue
- Interest points (I tend to write a lot of `HEREEEEEE!!!: 1`, `HEREEEEEE!!!: 2`, etc)
- Anything that is mildly interesting 🤷🏻‍♂️

### 3. Reduce search area, clean and repeat

Once you are pretty sure that something is not related to the issue, you can remove those log statements.

For example, if you see in the command line `HERE!: 1` and `HERE!: 2`, but not `HERE!: 3` then you are 100% that something's failing between log 2 and 3.

You can now `cmd+click` whatever is there and start over from step 1.

## Why not use the debugger?

I tend to use the debugger when:

- I need to gain insights into something that I'm already familiar with
- I got the issue narrowed down to a small portion of the code and still can't figure it out
- The code it's linear and doesn't involve any threads/goroutines
- The code being debugged doesn't span across multiple dependencies

> If the debugger is a sniper rifle then my process above is a machine gun. With the sniper rifle you know who your target is, while with the machine gun we are just shooting in the dark hoping to hit something.

Counter Strike analogy aside, my process = broad debug, debugger = narrow debug.

Also in Go's case, it's faster to write log statements and re-run the code than to wait for the debugger to initialize every time you make a change (or maybe I have a shitty computer).