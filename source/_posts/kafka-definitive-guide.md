---
layout: post
title: "kafka-definitive-guide"
subtitle: "keep reading and loving"
draft: false
date: 2021-09-30 12:19:34
categories: books
tags: Kafka
permalink:
description:
cover_img: copyright-free-images.jpg
toc-disable:
comments: true
---


<p align="center">The goal of A-RSnippet theme is to be comprehensive.</p>

<div align="center">
<a href="https://github.com/huyingjie/hexo-theme-A-RSnippet/tree/master" target="_blank"><img src="https://travis-ci.org/huyingjie/hexo-theme-A-RSnippet.svg?branch=master" style="display:inline"></a> <a href="https://discord.gg/CB6CPzq" target="_blank"><img src="https://img.shields.io/discord/405912462031060992.svg" style="display:inline"></a> <a href="http://hexo.io" target="_blank"><img src="https://img.shields.io/badge/hexo-%3E%3D%203.0-blue.svg" style="display:inline"></a> <a href="https://github.com/huyingjie/hexo-theme-A-RSnippet" target="_blank"><img src="https://img.shields.io/badge/Release-v0.1.0-red.svg"></a> <a href="https://github.com/huyingjie/hexo-theme-A-RSnippet/blob/master/LICENSE" target="_blank"><img src="https://img.shields.io/badge/license-GPL3-pink.svg" style="display:inline"></a></div>

# Chapter 1 : Meet kafka
Almost every company is powered by data, the less effort we spend on moving data around, the more we can focus on the core business at hand. It' a reason reason why we need Kafka.


# Chapter 5: Kafka internals
This chapter introduces how Kafka replication works, request interaction between consumer & producer, storage methods in file format & indexes.

## Cluster Membership
- Kafka use Zookeeper to maintain the list of brokers.
- Every broker registers itself with it's unique ID in Zookeeper by creating ephemeral node. Therefore, Kafka components like consumer or producer can watch subscribed broker's path in Zookeeper to get notify.
- If a broker get crash, the ephemeral node that the broker crated will be automatically removed. Due to the replication, it can later safety recover.

## Controller
- The Controller is one of the Kafka brokers, usually is the first broker starts in the cluster. Others broker create Zookeeper watch on this controller node to get notification. New Controller will be created when the old one get crash. All partitions that had a leader on that crashed one need a new leader.
- In summarize, the controller is responsible for electing leaders among the partitions and replicas whenever it notice nodes join or leave the cluster. The controller use epoch number to prevent "split brain" scenario.

## Replication
- As we said before, replication is a critical method to guarantee the availability and durability of Kafka.
-  Data in Kafka is organized by topics. Each topic is partitioned, and each partition can have multiple replicas. Those replicas are stored on brokers, and each broker typically stores hundreds or even thousands of replicas belonging to different topics and partitions.
- Two types of replica:
    - **Leader**: All produce and consume requests go through the leader replica.
    - **Follower** replicate messages from the leader and stay up to date. New Leader will be elected form follower if old leader get crash.
- **syn**: in order to maintain the message order, replica send the leader fetch request. The fetch request contains offsets to prevent out of syn. Therefore, the leader knows each replica's offsets.
- Only in-sync replicas are eligible to be elected as partition leaders in case the existing leader fails.

## Request Processing
- Most of what a Kafka broker does is process requests sent to the partition leaders from clients, partition replicas, and the controller
- All requests have a standard header that include:
  - Request types
  - Request version
  - Correlation ID (for errors log)
  - Client ID
- Two common types of requests are:
  - Produce requests: Sent by producers and contain messages the clients write to kafka
  - Fetch requests: Sent by consumers and follower replicas when they read messages from Kafka brokers.
- All produce and fetch requests must be sent to the leader replica. The broker received requests should contains the leader for the relevant partition for the request. Therefore, another request called a **metadata request** can be used, which contains partition list in the interested topics, the replica for each partition, and leader replica's ID.
- Metadata is cached in every brokers. Client will also maintain this metadata by **metadata request**, and refresh it after "Not a Leader" error occurs.

### Produce request
running as following steps, if the broker that contains the lead replica for a partition receives a produce request.
1. Does the user sending the data have write privileges on the topic?
2. number of acks specified in the request valid 0? 1? all?
3. If acks is set to all, are there enough in-sync replicas for safely writing the message?
4. Writing the new message to local disk

### Fetch request
1. partition leader checks if the request is valid (offset exists)
2. broker read messages from partition, send to the clients by zero-copy method.
3. Note that only the messages are cached in all in-sync replicas will be sent. Consumers only see messages that were replicated to in-sync replicas.

## Physical Storage
### Partition Allocation
- Kafka first keeps these rules to execute partition allocation within brokers:
  1. Spreading replicas evenly among brokers
  2. For each partition, each replicas is on a different broker
  3. assigning the replicas for each partition to different racks if possible (in Kafka release 0.10.0 and higher)
- Kafka use round-robin manner to determine the location for the leaders.

### File Management
- Kafka use retention period for each topic to implement the retention concept.
- Due to the huge time-consuming of message finding and purging in a large file, Kafka split each partition into several segments. A segment has limit time and size of data.
- Each segment is stored in a single data file

### File format
- It's recommended to use compression on the producer which will benefit for sending huge messages.
- sending larger batches means better compression both over the network and on the broker disks.
