---
title: 「读书笔记」RabbitMQ 实战指南前四章笔记
author: 陈加菲
date: 2019-06-16 13:38:47
tags:
 - 读书笔记
 - RabbitMQ
---

这一个月读了《[RabbitMQ 实战指南](https://book.douban.com/subject/27591386/)》，自觉对 RabbitMQ 的认识比以前更深了一点，更系统了一点。

这篇文章主要记录了我在阅读前四章中整理的思维脑图。前四章的知识点，涵盖了 RabbitMQ 日常应用的大部分知识点，有必要系统的梳理一遍。

### 01 RabbitMQ 简介

第一章简单介绍了相关概念，例如消息中间件，RabbitMQ 的含义。这是入门性质的概念，读多几遍，思考这种技术提出来要应对的问题，其提供的解决方案，便容易深入理解。

<img src="https://jluncc-blog.oss-cn-hangzhou.aliyuncs.com/images/rabbitmq-reading-notes-1/01-RabbitMQ.png" alt="01 RabbitMQ 简介" title="01 RabbitMQ 简介" style="zoom:1000%;" />

### 02 RabbitMQ 入门

第二章继续深入介绍 RabbitMQ 的核心概念，例如交换器、队列、绑定关系等，理解好这些核心概念，后面消息的发送接收等操作，不同特点的队列，便更容易掌握。同时这里也介绍了底层使用的 AMQP 协议。

<img src="https://jluncc-blog.oss-cn-hangzhou.aliyuncs.com/images/rabbitmq-reading-notes-1/02-RabbitMQ.png" alt="02 RabbitMQ 入门.png" title="02 RabbitMQ 入门.png" style="zoom:100%;" />

### 03 客户端开发向导

在第三章里，详细介绍了生产者发送消息，到消费者消费消息的一整个流程，以及其中需要注意的小细节等，这一章较为重要，要加深理解。

<img src="https://jluncc-blog.oss-cn-hangzhou.aliyuncs.com/images/rabbitmq-reading-notes-1/03-RabbitMQ.png" alt="03 客户端开发向导.png" title="03 客户端开发向导.png" style="zoom:100%;" />

### 04 RabbitMQ 进阶

第四章的进阶知识，是在第三章的基础上的。介绍了 TTL、DLX 等特性，以及生产者确认机制，持久化机制等等。这章里要理解一个重点，就是保证消息传输的可靠性，通俗点讲就是**保证消息的 0 丢失**。

如何保证消息的 0 丢失呢，主要有 4 点注意

1. 消息生产者需要开启事务机制或 publish confirm 机制，确保消息可以可靠传输到 RabbitMQ 中
2. 消息生产者需要配合使用 mandatory 参数或者备份交换器来确保消息能够从交换器路由到队列，进而能够保存下来而不会被丢弃
3. 消息和队列都需要进行持久化处理，确保 RabbitMQ 服务器在遇到异常时不会造成消息丢失
4. 消费者在消费消息同时需要手动应答，确认已正确消费消息，避免在消费端引起不必要的消息丢失

<img src="https://jluncc-blog.oss-cn-hangzhou.aliyuncs.com/images/rabbitmq-reading-notes-1/04-RabbitMQ.png" alt="04 RabbitMQ 进阶.png" title="04 RabbitMQ 进阶.png" style="zoom:100%;" />

**END**