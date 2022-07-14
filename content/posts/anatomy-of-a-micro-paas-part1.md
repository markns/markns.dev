+++
author = "Mark Nuttall-Smith"
title = "Anatomy of a micro PaaS - part 1"
date = "2022-07-08"
description = "How to build a micro PaaS using Kubernetes and Istio"
tags = [
    "architecture",
    "kubernetes",
    "PaaS",
]
draft = false
+++

Languages are amazing things.
For all the beauty of music and dance, neither has the ability to convey information as precisely as human language.
Sure, there's few better expressions of romantic love than Percy Sledge's [When a Man Loves a Woman](https://www.youtube.com/watch?v=EYb84BDMbi0), or more convincing statements of intergalactic dominance than an enthusiastic performance of [the Hammer dance](https://www.youtube.com/watch?v=keAhk3Lz6E8&t=34s).
But if you need to give someone directions to the railway station, or explain the poor numbers of the sales unit in the quarterly report, you're _probably_ better off using a human language rather than humming or interpretive dance. 

As it is with human to human communication, so it is with human to computer communication.
Point and click user interfaces, and low-code platforms are extremely capable these days. 
And who would argue against the spreadsheet being the most important software application of all time (ok, web-browsers we see you, please sit down)?.
However, in some scenarios, when we need to direct a computer in _precisely_ the manner we desire, there is no substitute for the expressiveness and fidelity of a programming language.

```python
# If you want to tell your computer you love him/her/it, 
# you will need a programming language
while True:
    i.love(you)
```

Learning some basic programming is also not that hard. 
A language like Python has a syntax similar to English, and there are a huge variety of high-quality resources online to help beginners start coding.

Learning the entire discipline of software engineering, on the other hand, can be a life's work.
Non-functional requirements such as security, monitoring, version control, build systems, logging, dependency management, deployment, resource isolation ([and more!](https://en.wikipedia.org/wiki/Non-functional_requirement#Examples)) can easily turn one hour of coding into months of enterprise bureaucracy.

## Enter the PaaS 

The promise of PaaS's such as Heroku or OpenShift is to increase productivity by enabling engineers to focus on the _development_ of an application. 
They achieve this by standardising and automating as many non-functional requirements as possible, thereby reducing complexity and lead times.
If you're a developer used to emailing IT to obtain a VM/database on which to run your application and waiting hours/days/weeks for it to be delivered, the empowerment you will feel when handed the keys to a PaaS is priceless.   

However, for the most part, you do still need to be a developer to use these systems. 
Because the intention is to constrain the user as little as possible, they leave a lot of choices to be made during development. 
They also require familiarity with the command line, git, databases and other tools normally outside the skill-set of non-developers.

For this reason, over the last few years, a new generation of hyper-focused domain-specific micro-PaaS's have been brought to market.
These systems enable a user to write a tiny amount of code - perhaps just the body of a function - directly in a browser based IDE.
The user code either has pre-configured access to domain-specific APIs, or may be written in a Domain Specific Language (DSL) itself.

There are many examples in the crypto space - for example, you can use [Trality](https://www.trality.com/creator/code-editor) to create managed trading bots, and [Remix](https://remix-project.org/) to develop and deploy smart contracts for Ethereum.
In more traditional finance, [QuantConnect](https://www.quantconnect.com/) is a platform that can be used for backtesting and live-trading automated strategies using Python and C#.
In the IoT world, the [ThingsBoard](https://thingsboard.io) rules engine allows you to write JavaScript to react to incoming events. 
Something more fun, but no less valuable - [Code Combat](https://codecombat.com/) is a game-based learning platform that teaches kids how to code.

The rise of tools such as Kubernetes and VSCode-for-the-Web has reduced the cost of developing a micro-PaaS dramatically, creating opportunities that wouldn't have been economically viable just a few years ago. 
The remainder of the posts in this series will describe _one possible_ architecture of such a system.

The [next post]({{< relref "/posts/anatomy-of-a-micro-paas-part2.md" >}}) continues with the example use case and presents the platform at a high level. 

