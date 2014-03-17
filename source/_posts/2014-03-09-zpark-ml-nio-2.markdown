---
layout: post
title: "ZPark-Ztream II (Part 2/3): Fancy Spark Streamed Machine-Learning & new Scalaz-Stream NIO API"
date: 2014-03-09 19:19
comments: true
external-url:
categories: [scala,spark,scalaz-stream,NIO,machine learning,stream,map reduce,ML,recommendation]
keywords: scala,spark,scalaz-stream,NIO,machine learning,stream,map reduce,ML,recommendation
---

## Synopsis

The code & sample apps can be found on [Github](https://github.com/mandubian/zpark-ztream)

> [Zpark-Zstream I article](/2014/02/13/zpark/) was a PoC trying to use [Scalaz-Stream](https://github.com/scalaz/scalaz-stream) instead of DStream with [Spark-Streaming](https://spark.incubator.apache.org/). I had deliberately decided not to deal with fault-tolerance & stream graph persistence to keep simple but without it, it was quite useless for real application...

<div class="well">
<p>Here is a tryptic of articles trying to do something <i>concrete</i> with <a href="https://github.com/scalaz/scalaz-stream">Scalaz-Stream</a> and <a href="https://spark.incubator.apache.org/">Spark</a>.</p>
<p>So, what do I want? I wantttttttt <i>a shrewburyyyyyy</i> and to do the following:</p>
<ol>
<li>Plug Scalaz-Stream Process[Task, T] on Spark DStream[T] (Part 1)</li>
<li><b>Build DStream using brand new Scalaz-Stream NIO API (client/server) (Part 2)</b></li>
<li>Train Spark-ML <i>recommendation-like</i> model using NIO client/server (Part 3)</li>
<li>Stream data from multiple NIO clients to the previous trained ML model mixing all together (Part 3)</li>
</ol>
</div>

<br/>
# [Part 2/3] From Scalaz-Stream NIO client & server to Spark DStream

## Coming very soon

<br/>
<br/>
<br/>
<br/>

