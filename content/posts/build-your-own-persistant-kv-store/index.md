---
title: "Build Your Own Fast, Persistant KV Store"
description: ""
date: 2023-01-03T18:15:37+04:00
draft: false
categories: [databases]
tags: [key-value, performance, bitcask, diy]
---

Databases have always fascinated me, and I have always dreamt of building a simple, hobby database for fun.
I have read several blog posts about building redis, git, compiler, and interpreter, but none about building a database. This post is an outcome of a long and treacherous search of posts on making a database and eventually encountering the [Bitcask paper](https://riak.com/assets/bitcask-intro.pdf).

## Bitcask

[Bitcask](https://riak.com/assets/bitcask-intro.pdf) is a local key/value store where the data is persisted in a file. One operating system process will open the file for writing at any given time. When the file size reaches a considerable theoretical threshold, we close the file and open a new one.

### Goals

1. Low latency per item read or written.
2. High throughput, especially when writing an incoming stream of random items.
3. Ability to handle datasets much more significant than RAM w/o degradation.
4. Crash friendliness, both in terms of a fast recovery and not losing data.
5. Ease of backup and restore.
6. A relatively simple, understandable code structure and data format.

### Data format


<img src="images/bitcask-db.svg" style="border:none;" alt="Data Format"/>
