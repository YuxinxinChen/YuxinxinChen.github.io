---
layout: post
title: "Mini Gunrock Reading Notes"
data: 2017-10-10
tags: [reading notes, GPU]
comments: true
share: false
---

### Graph library system models: BSP, AP, GAS



### Pull-based model VS Push-based model

Pull-based model: When data exchanges needed between two or more threads, processor, machines, in pull-based approach, one thread will try to access the data another threads is working on (processor access data another processor working on, or one machine send an mem access request on network to another machine). Because it might pull inconsistent data from any concurrently executing neighbours, pull-based asynchronous implementation need to use lock to ensure data inconsistency.Gather, Apply and Scatter (GAS) model is pull-based and its asynchronous implementation uses distributed locking to ensure data consistency.

Push-based model: The information is sended explicitly after finishing the computation related to that piece of information. Then the receiver receive the consistent information passively. However it requires the sender push the message explicitly. Since messages are buffered in a local message store, concurrent reads and writes to the store can be handled locally with local locks or lock-free data structure. 
