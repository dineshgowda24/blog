---
title: "Pgo: Writing a Postgres Driver from Scratch in Go"
date: 2024-09-18T23:19:00+05:30
draft: true
---

## Preface

A couple of weeks back, I was thinking about benchmarking libpq and pgx driver to understand the performance bottlenecks in both drivers as in our production set up we have been using libpq and I read about Phil's post about pgx being better for production use.

One thing that started bugging me was why do we have two drivers in the first place, I felt that low level communications are trivial and there should have been only drivers. Another thing that also confused me was redis client in go does not have a seperate driver, then what's special about Postgres, which is when I discovered Postgres Wire Protocol.

I started reading about the protocol itself and so many things started to resonate with me and things started becoming clearer. I loved the protocol and eventually decided to write a toy driver to understand lower level workings of the protocol itself.

As decided to write a toy driver, I did not have any idea about how to start, so I started reading about the protocol itself and started writing a simple client to connect to the server and send a simple query. Since I have no prior experience in writing a driver, I did go through the source code of libpq and pgx to understand how they are implemented and how each component within the driver is implemented.

Over the course of time, I did add a flavour of my own to the driver, since performance was not my primary goal, I focused on making it easier to read and follow along.

## Tld'r

If are someone who already has a understanding of how the protocol works and want to see the driver, you can directly jump to the [*Github repository*](https://github.com/dineshgowda24/pgo).

## Postgres Wire Protocol

The protocol itself is very simple and easy to understand, it is a TCP based protocol. Every data that is sent or received is a type of message. There are two types of messages. A frontend message and a backend message. A frontend message is sent by the client(our Go driver) to the server(Postgres) and a backend message is sent by the server(Postgres) to the client(our Go driver).

**Message Format**

Every message starts with a single byte message type, followed by a 4 byte message length(including the length of the message type) and the message body. Except for one message called `StartupMessage`, all other messages are in the following format.

```
+----------------+----------------+----------------+
| 1 byte         | 4 bytes        | n bytes        |
+----------------+----------------+----------------+
| Message Type   | Message Length | Message Body   |
+----------------+----------------+----------------+
```