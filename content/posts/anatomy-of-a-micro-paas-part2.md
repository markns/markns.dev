+++
author = "Mark Nuttall-Smith"
title = "Anatomy of a micro PaaS - part 2"
date = "2022-07-12"
description = "How to build a micro PaaS using Kubernetes and Istio"
tags = [
    "architecture",
    "kubernetes",
    "PaaS",
]
draft = false
+++

In [part 1]({{< relref "/posts/anatomy-of-a-micro-paas-part1.md" >}}) of this series we discussed why and when a micro-PaaS can be useful. 
This post will present a specific micro-PaaS use case.
Part 3 will show how we can build our platform by customizing Kubernetes through its native extension points. 

## But first, dancing Bill Gates.

Real time data is not the first thing that comes to mind when you think of Microsoft Excel, but in fact Excel has some impressive capabilities in this area.
Anyone who has worked on a trading floor knows how common the use of Bloomberg Excel Add-in is there.
Amongst other things, this add-in can stream live data from Bloomberg's APIs directly into users spreadsheets.
It's not unreasonable to say that billions of dollars of financial activity rely on this tool on a daily basis.

A previous company of mine also produced an Excel add-in that connected to Python applications running in the cloud.
The Python apps could grab data from anywhere - APIs, event-streaming platforms, databases - and push it directly to the Excel spreadsheet in real-time.
Once upon a time we [hit the front page of Reddit](https://www.reddit.com/r/dataisbeautiful/comments/8ddmui/real_time_stock_dashboard_in_excel_oc). 

We also managed to stream a grooving Bill Gates into Excel. 

![epic dance](/images/epic-dance.gif)

If you want to know how it works, watch [this video](https://www.youtube.com/watch?v=UBX2QQHlQ_I) and imagine that the RGB values are being streamed in a loop into the spreadsheet. Alright, I know what you're thinking: 🤯, but what does this have to do with a micro-PaaS exactly? 

## The value proposition

Let's stop thinking about how cool Bill Gates is for a moment, and discuss our value proposition. 

Imagine we're a company with a data backbone built on Apache Kafka (and [who isn't](https://kafka.apache.org/powered-by) these days), and many business users who are using Excel (also not hard to imagine I guess).

We want to be able to easily connect the Excel users with the Kafka data, with small transformations as the data is in-flight. We want multiple users to be able to connect to the same connector, fanning-out the data feed. 
We want the platform to be accessible to non-developers, so the connectors should be written in Python, as this the [most popular](https://statisticstimes.com/tech/top-computer-languages.php) and user-friendly language available. 
There should be a simple API for consuming data from Kafka, and a simple API for streaming data to Excel.
Let's call the platform AeroGrid. 

Here's how a simple user app for summing order quantities by product name might look:

```python
import aerogrid

# the application object provides the core API   
app = aerogrid.App('ordersapp', broker=BROKER_URI)

# Records describe how messages in Kafka are serialized:
# for example: {"product": "How to....", quantity": 300}
class Order(aerogrid.Record):
    product: str
    quantity: int

# connect to the orders topic. key is order_id
orders_topic = app.topic('orders', key_type=str, value_type=Order)

# the grid object is used to send data to Excel 
product_sum = app.Grid('product_sum', columns=['units_sold'], 
                       default=lambda: {'units_sold': 0})

@app.stream(orders_topic)
async def sum_orders(orders):
    async for order_id, order in orders.items():
        units_sold = product_sum[order.product]['units_sold'] + order.quantity
        product_sum[order.product] = { 'units_sold': units_sold }
```

For a user to connect to the system, they can use the `AeroGrid` function installed by the Excel Add-in.
The function takes the name of the app (`ordersapp`) and grid (`product_sum`) as arguments, so the request can be routed correctly.
This is how it might look:

![product sum](/images/product_sum.gif)

We won't dig too much further into the architecture of the core product, since it will be different for every micro-PaaS.
But let's just say that this one could be built using [aiokafka](https://github.com/aio-libs/aiokafka) and a [gRPC streaming server](https://grpc.io/docs/what-is-grpc/core-concepts/#server-streaming-rpc) on the Python side, and the fantastic [Excel-DNA](https://github.com/Excel-DNA/ExcelDna) library on the client side for the add-in.

Here's a sketch:

![paas core](/images/paas-core.png)

This diagram shows the most simple case - one user connected to one app, but in reality we want to enable:

- Multiple users to be able to connect to the same app
- Multiple apps available to a user
- Multiple platform tenants - either teams or customers

There's some more work to be done!

## P is for Platform

Most obviously, we need a control plane for the system, so users can create, edit, start, stop and delete apps.
We also need a mechanism for clients to receive the list of the apps running in the platform, together with their status.

The diagram below shows the AeroGrid platform from the perspective of a single tenant (group of users).
There are two apps in the system, shaded in orange.
App 2 is already running and sending data from Kafka to connected Excels.
The control flow (black arrows) show a user starting App 1 from the AeroGrid web console. 
This action causes App 1 to be deployed and start running on the AeroGrid infrastructure.
Finally, when App 1 is completely ready, the service discovery component notifies the Excel add-ins that a new app is running and can be connected to if desired.

![paas](/images/paas.png)

In principle, this control flow is the same for other actions in the system.

There are still quite a few open questions. 
For example, how do connections from Excel get routed to the correct apps?
How do we identify users and ensure they are authorized to perform the actions they attempt? 
How do we make sure the behaviour of one tenant in the system doesn't impact the experience of another tenant - either maliciously or accidentally?
How do we onboard new tenants into the platform? 

In the next part of this series we will answer these questions using the wonders of Kubernetes, Istio and Auth0.