---
title: "Supercharging GCP Navigation with Alfred"
date: 2025-06-05T23:04:42+04:00
draft: false
categories: [productivity]
tags: [alfred, google cloud platform, productivity, go]
images: ["/posts/supercharging-gcp-navigation-with-alfred/images/demo.gif"]
---

Navigating Google Cloud Platform, or any cloud platform for that matter, can be draining, primarily when you work with cloud services every day. If you manage multiple projects, just accessing the right resource can feel like a nightmare because of the number of clicks involved.

## TL;DR

If you want to skip the story and go straight to the workflow, you can find it on alfred-gcp-workflow[^1]


<img src="images/demo.gif" width="80%"  alt="Demo" />

## Preface

We heavily utilize GCP at work, and I also use it for my personal projects. Typically, it takes at least 5 to 6 clicks to reach the desired location. It's not a huge deal on its own, but when you do this several times a day, it becomes a frustrating time-sink. It breaks your flow, slows you down, and forces you to reach for the mouse more often.

{{<notice tip>}}
🧠 Even a 15-second task repeated 10 times daily is over 15 minutes per week.
{{</notice>}}

I've always disliked using the mouse, so I started looking for a way to streamline GCP access. One of the best decisions I made a few years ago was switching to Alfred[^2] as my main launcher. Since then, I've built several custom workflows (not open-sourced) to streamline daily tasks, including JWT decoding, timestamp conversions, and more. It surprises me that many senior engineers still use browsers for everything. That's fine, but it wastes time and energy.

## Why I Built the Workflow

If you've worked in an organization with many projects and high traffic, you know how clunky GCP console navigation can get. It's even worse during incidents. When you're on call and alerts are flying in, the last thing you want is to fumble through tabs to locate the right resource while others in the incident bridge are waiting on you.

I had used the AWS workflow[^3] in my previous job and loved it. It provided me with quick access to AWS resources with minimal friction. [*I even contributed to it at one point*](https://github.com/rkoval/alfred-aws-console-services-workflow/commit/43704faccc4152fc6bd5066ca9b3a061ade61d2c). But for GCP, I couldn't find a solid equivalent, so I decided to build my own.

## From Idea to Implementation

<img src="images/idea.gif" width= "40%" style="border:none;" alt="idea"/>

I first thought of building this over a year ago. I assumed I could follow the same model as the AWS workflow, but I quickly realized GCP is an entirely different maze to navigate.

### Where GCP Made Things Harder

The AWS SDK is well-documented, centralized, and supports profile-based authentication. Switching between profiles in SDK is easy.

While GCP uses configurations similar to AWS profiles, its SDKs don't support config-based authentication. That made the whole thing messy. It initially blocked me, and I shelved the idea for a while.

### The Aha Moment!

Months later, it hit me. Why not use the `gcloud`[^4] CLI?

It supports config-based authentication and it exposes a wide set of commands for listing resources. I tried a few things and quickly realized it could work. Funny how it had been right in front of me the whole time.

Once I committed to this new direction, the implementation took just a few weeks.

## Key Features of the Workflow

After sitting on the idea for months, it came together quickly. Some highlights:

- Quick setup: install it, set your `gcloud`[^4] path, and you're ready
- Access to **250+** GCP services like Compute Engine, Cloud Storage, BigQuery, and more
- Support for **20+** resource types, including projects, instances, and buckets
- Fuzzy search for fast lookup
- Smart caching to speed up access
- Config and region switching with `@` and `$`
- Developer tools to manage workflow state, cache, and updates

## Real-World Use Cases: How does it improve my productivity?

### Incidents Response

It's late at night, and I'm oncall[^5]. An alert indicates that the SQL instance CPU is spiking or a Pub/Sub subscription ACK is lagging. I need to check GCP right away. Instead of opening multiple tabs and clicking through the console, I can type `gcp compute instance` or `gcp pubsub` and jump right in.

### Everyday Tasks

Need a link to a storage bucket? Just type `gcp storage bucket` and copy the URL. No need to remember where it is in the console. These small moments add up.

### Less Cognitive Overhead[^6]

The workflow eliminates the mental overhead of remembering where things are in GCP. I type what I want and go. It feels like having a fast lane through the chaos.

## Lessons Learned

This workflow taught me a lot about the `gcloud`[^4] CLI, the Alfred ecosystem, and how to create tools that actually save time. 

Sometimes, the simplest solutions are the best. They take time to surface. It's also okay to step away from an idea and revisit it later with fresh eyes. What felt blocked before might suddenly feel obvious.

It also taught me that manifesting ideas takes time, patience, and a willingness to adapt.


## Try It Out

If this sounds useful, give it a spin: alfred-gcp-workflow[^1]. I'd love to hear your feedback, ideas, or bug reports. Feel free to open issues or suggest features.

*Let's make cloud navigation painless. ♥️*

[^1]: https://github.com/dineshgowda24/alfred-gcp-workflow
[^2]: https://www.alfredapp.com
[^3]: https://github.com/rkoval/alfred-aws-console-services-workflow
[^4]: https://cloud.google.com/cli
[^5]: https://sre.google/sre-book/being-on-call/
[^6]: https://www.psychologs.com/cognitive-overload/