---
layout: post
title:  "Airflow Celery distributed processing"
date:   2021-08-16 21:13:37 +0700
categories: distributed-processing
tags: celery airflow
tag: distributed-processing
---

# Celery
[Introduction to Celery][Introduction to Celery]  

Task queues are used as a mechanism to distribute work across threads or machines.

A task queue’s input is a unit of work called a task. Dedicated worker processes constantly monitor task queues for new work to perform.

Celery communicates via messages, usually using a broker to mediate between clients and workers. To initiate a task the client adds a message to the queue, the broker then delivers that message to a worker.

A Celery system can consist of multiple workers and brokers, giving way to high availability and horizontal scaling.

Celery is written in Python, but the protocol can be implemented in any language. In addition to Python there’s node-celery and node-celery-ts for Node.js, and a PHP client.

Language interoperability can also be achieved exposing an HTTP endpoint and having a task that requests it (webhooks).

Celery has the ability to communicate and store with many different backends (Result Stores) and brokers (Message Transports).  

Backend can use: Redis, RabbitMQ, SQLAlchemy  
Broker can use: Redis, RabbitMQ, AWS SQS  

# Monitor Celery system
[Flower - Celery monitoring tool][Celery Flower]  

![Broker monitor](/rin-rin-blog/assets/images/distributed/celery/flower.png){: width="500"}  

# Airflow architecture

![Airflow architecture in Google cloud](/rin-rin-blog/assets/images/distributed/celery/airflow-architecture.png) 

[Introduction to Celery]: https://docs.celeryproject.org/en/latest/getting-started/introduction.html#what-s-a-task-queue
[Celery Flower]: https://flower.readthedocs.io/en/latest/