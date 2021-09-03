---
layout: post
title: "How to fix the Excel VBA memory leak"
date: 2021-09-03 22:25:00 +0200
tags: [excel, vba, garbage collection]
categories: Programming
---

## The problem
The gist of the matter is that some Excel macros started causing extreme memory leaks when used in version 2013 and later. My first searches online yielded results instructing to avoid copying and pasting, to disable calculations for the duration of the macro etc. I've done all I possibly could based on those results, but the issue persisted.

## The solution
I finally came across the solution to set all declared and used objects to `Nothing` after they are no longer in use.
This indeed works, simple as that. The important thing is to set them to `Nothing`in the reverse order they were set at
the beginning.


{% include comments.html %}