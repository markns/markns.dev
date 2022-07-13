+++
author = "Mark Nuttall-Smith"
title = "Architecture of a micro PaaS - part 2"
date = "2022-07-12"
description = "How to build a micro PaaS using Kubernetes and Istio"
tags = [
    "architecture",
    "kubernetes",
]
draft = false
+++

In [part 1]({{< relref "/posts/architecture-of-a-micro-paas-part1.md" >}}) of this series we discussed why and when a micro-PaaS can be useful. 
This post will present the example use case for the rest of the series. 
Revise: and start to discuss how we can build a micro-PaaS by customizing Kubernetes using its native extension capabilities. 

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

If you want to know how it works, watch [this video](https://www.youtube.com/watch?v=UBX2QQHlQ_I) and imagine that the RGB values are being streamed in a loop into the spreadsheet. Alright, I know what you're thinking: 🤯, but what does this have to do with micro-PaaS's exactly? 

## The value proposition

Let's stop thinking about how cool Bill Gates is for a moment, and discuss our value proposition. 

Imagine we're a company with a data backbone built on Apache Kafka (and [who isn't](https://kafka.apache.org/powered-by) these days), and many business users who are using Excel (also not hard to imagine I hope).

We want to be able to easily connect the users with the data, with small transformations as the data is in-flight. We want multiple users to be able to connect to the same connector, fanning-out the data feed. 
The connectors should be written in Python, as this the [most popular](https://statisticstimes.com/tech/top-computer-languages.php) and user-friendly language available. 
There should be a simple API for consuming data from Kafka, and a simple API for streaming data to Excel.
Let's call the platform AeroGrid. 

Here's how a simple user app for summing order quantities by product name might look:

```python
import aerogrid

# the application object provides the core API   
app = aerogrid.App('ordersapp', broker=BROKER_URI)

# Records describe how messages in Kafka are serialized:
# for example: {"product": "How to....", amount": 300}
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

And this is how it might look to a user in Excel:

![product sum](/images/product_sum.gif)

We won't dig too much into the architecture of the core product, since it will be different for every micro-PaaS, but let's just say that this one could be built using [aiokafka](https://github.com/aio-libs/aiokafka) and a [gRPC streaming server](https://grpc.io/docs/what-is-grpc/core-concepts/#server-streaming-rpc) on the Python side, and the fantastic [Excel-DNA](https://github.com/Excel-DNA/ExcelDna) library on the client side for the add-in.

Here's a sketch:

![paas core](/images/paas-core.png)

This diagram shows the most simple case - one user connected to one app, but in reality we want to enable:

- Multiple users connected to the same app
- Multiple apps available to a user
- Multiple user groups - either teams or customers

We also need a control plane for the system, so users can create, edit, start, stop and delete apps. 

## P is for Platform

