---
comments: true
title: Outbox pattern in Quarkus
draft: true 
date: 2024-01-29
authors:
  - mcruzdev
categories:
  - Java
  - Quarkus
  - Microservices
  - Pattern
  - Panache
  - DevServices
  - Scheduler
---

If you need to save data into the database and then notify another service about the local changes, this post is for you!

In this post, we will explore how to solve dual writes in a distributed systems using Quarkus.

## Scenario

Imagine you have an order system, and every time an order is created or its status is changed, you need to save the details of that order to generate custom reports.


## Doing it with a monolith

In a monolith it is very simple, I will do:

1. Create the `order`
2. Save the `order` into the database;
3. Save the order details in the database for reporting purposes;

I can do all in a single transaction, right?

<!-- more -->

```java
public CreateOrderOutput execute(final CreateOrderInput input) {
    Order order = new Order(); // 1. Create the order

    QuarkusTransaction.requiringNew().run(() -> {
    // 2. Save the `order` into the database;
    // 3. Save the order details in the database for reporting purposes;
    })

    return new CreateOrderOutput(order);
}
```

Either everything happens, or nothing does—atomicity at work. Working with a monolith is straightforward; you don't have to concern yourself with the complexities of distributed transactions that often arise in microservices architectures. In a monolithic system, everything is contained within a single codebase, making it easier to manage transactions and ensure consistency across the application.

## Doing it with Microservices

Let's consider a scenario with two microservices: **order-service** and **report-service**. The **order-service** receives an HTTP request, saves the order, and it's necessary to notify the **report-service** when the order is created. We will see how to achieve this goal with code samples.

### Code 1

```java linenums="1"
public CreateOrderOutput execute(final CreateOrderInput input) {
    Order order = orderRepository.findById(input.getOrderId());
    
    Order order = new Order();
    
    publisher.send(new OrderCreatedEvent(order));

    QuarkusTransaction.requiringNew().run(() -> {
        orderRepository.save(order);
    });

    return new CreateOrderOutput();
}
```

It's incredible! We are utilizing a queue, using an event-driven architecture woooow! But, with great power comes great responsibility!

**What is the problem here? tic! tac! tic! tac! :clock:**

**Answer:** We are publishing to the queue first, if the database fails the **report-service** will get a incosistent data.

### Code 2

Alright, now I do everything in a single transaction: first, I try to save it in the database, then I send it to the queue. If the queue fails, I don't save it in the database.

```java linenums="1"
public CreateOrderOutput execute(final CreateOrderInput input) {
    Order order = orderRepository.findById(input.getOrderId());
    
    order.changeStatus(input.getNewStatus());
    
    QuarkusTransaction.requiringNew().run(() -> {
        orderRepository.save(order);
        publisher.send(new OrderCreatedEvent(order));
    });

    return new CreateOrderOutput();
}
```

**What is the problem here? tic! tac! tic! tac! :clock: :clock:**

**Answer:** We are adding I/O operation into a database transaction, it is a wrong decision and bad practice.

??? tip "Do not execute I/O operation into the transaction"
        
    The problem with putting I/O operations inside a transaction is that it can cause locking and increase waiting time since transactions typically lock resources until they are completed. Additionally, some I/O operations may not be transactional by nature, which can lead to unexpected behaviors or partially completed transactions in case of failure. Instead, it is generally preferable to perform I/O operations outside the transaction or in a separate transaction, depending on the specific requirements of the system.


### Code 3

Ok, I will try again...

```java linenums="1"
public CreateOrderOutput execute() {

    Order order = new Order();

    QuarkusTransaction.requiringNew().run(() -> {
        orderRepository.save(order);
    });

    publisher.send(new OrderCreatedEvent(order));

    return new ChangeOrderStatusOutput();
}
```

**What is the problem here? tic! tac! tic! tac! :clock: :clock: :clock:**

**Answer**: The problem here is that the publishing of the event can fail. If it fails, the **report-service** will not be notified.

## The Outbox pattern solution

When we perform multiple atomic actions at once, such as saving order data and sending a message, it's challenging to maintain consistency. The client cannot keep trying indefinitely.

In our situation, we are trying to **save order data** and **send a message** at the same time. To solve this, we need another task that **keeps trying until both actions are done**.

This other task, which will keep retrying until both actions are done, can be implemented using the **Outbox pattern**. In our scenario the Outbox pattern can be composed with two operations:

1. Save both the order and the message in the database. If the message is sent successfully to the queue, delete it from the database. If not, move to **step 2**.
2. Retry sending the message to the queue. We can do this thanks **step 1**.

Show me the code!

```java linenums="1"
public ChangeOrderStatusOutput execute() {
    
    Order order = new Order();

    OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(),
            order.getStatus().name(),
            order.getCreatedAt());

    Outbox outbox = new Outbox(eventToMessage(event)); // (1)

    QuarkusTransaction.requiringNew().run(() -> { // (2)
        this.orderRepository.persist(order);
        this.outboxRepository.persist(outbox);
    });

    this.publisher.send(event); // (3)

    QuarkusTransaction.requiringNew().run(() -> {
        this.outboxRepository.deleteById(outbox.getId()); // (4)
    });

    return Response.created(URI.create("/orders/" + order.getId())).build();
}
```

1. We are creating the `Outbox` instance using the `OrderCreatedEvent` object. Here, we will save the `OrderCreatedEvent` serialized.
2. We are saving both the `Order` and the `Outbox` within the transaction. This is necessary because I need to ensure that both are saved together.
3. We are publishing the `OrderCreatedEvent` to the queue.
4. if the publisher.send(event) works well, We will delete the `Outbox` record.

With this approach, we can ensure that we will not lose the `OrderCreatedEvent` event if the publishing step fails. If the publishing step fails, we have the `OrderCreatedEvent` message stored in the database through the `Outbox`. However, it is not complete yet we need to do the **step 2** - we have the event in the database, but we need to retry it because the **report-service** needs to be notified.

### Retry, retry, retry

Below, you can find a code sample where we are attempting to resend the message to the queue.


```java linenums="1"
@Scheduled(every = "5s") // (1)
void retry() {
    List<Outbox> outboxs = this.outboxRepository.listAll();
    List<String> ids = new ArrayList<>();

    for (Outbox outbox : outboxs) { // (2)
        this.publisher.send(outbox.getMessage());
        sent.add(outbox);
    }

    QuarkusTransaction.requiringNew().run(() -> { // (3)
        this.outboxRepository.delete("id in (?1)", ids);
    });
}
```

1. We are using Quarkus `scheduler` for scheduling periodic tasks.
2. For each `Outbox` entry, we are sending it to the queue and storing the result.
3. We are deleting all `Outbox` entries that have been sent within a single transaction.

!!! info 

    Adapt the code to fit your needs. Running it `every 5 seconds` is ideal for testing the Outbox retry functionality.

Note that our consumer (**report-service**) needs to be idempotent because there's a chance that the **order-service** might send the same message multiple times. Another crucial point to remember is that if I try to send an **OrderCreatedEvent** message and encounter an error, and then shortly after, the status of my order changes to **CANCELED**, there's a possibility that I might send the **CANCELED** event before the **order created event**.

## Considerations

In this post, you've seen how to implement the Outbox pattern. Another method for addressing the dual write issue in an asynchronous approach is by utilizing [Change Data Capture (CDC)](https://en.wikipedia.org/wiki/Change_data_capture). You can read more about CDC [here](https://en.wikipedia.org/wiki/Change_data_capture).

## Source code

I used the following technologies in the repository:

- Panache
- PostgreSQL
- DevServices
- Scheduler

If you'd like to view the entire code, you can access it [here](https://github.com/mcruzdev/quarkus-outbox-pattern).


## Thank you

That's all; thank you for reading! See you in the next post. Goodbye! 👋
