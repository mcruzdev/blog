---
comments: true
title: Outbox pattern in Quarkus
draft: false 
date: 2024-01-30
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

If you need to save data into two different systems, such as persist a data in the database and then notify another service about the local changes (through RabbitMQ, Kafka, ActiveMQ, etc.), this post is for you!

In this post, we will explore how to solve dual writes problem in a distributed systems using Quarkus.

## Scenario

Imagine you have an order system, and every time an order is created or its status is changed, you need to notify another system to perform an action. This scenario is common when working with microservices. In the microservice world, each microservice needs to be cohesive, have low coupling, and be deployable independently. With these principles in mind, the only way to communicate with another microservice is through the network, which often leads to an increase in dual writes. Dual writes involve writing data into two different systems simultaneously. For example, writing into the database and also writing into ActiveMQ.

In the upcoming sections, we will see this in action and look at some common mistakes people make when trying to fix the dual writes issue.

## Doing it with a monolith

In a monolith, it is very simple because most of the time we have just one write operation, where we are writing into one system (commonly database). Basically, what we need is:

I can do all things in a single transaction, right?

```java
public CreateOrderOutput execute(final CreateOrderInput input) {
    Order order = new Order(); // 1. Create the order
    
    // Create another domain object from the Order instance // (1)

    QuarkusTransaction.requiringNew().run(() -> {
        // 2. Persist the `order`;
        // 3. Save the order details in the database for reporting purposes;
    })

    return new CreateOrderOutput(order);
}
```

1. This other object, in a microservice architecture, would probably be in another microservice.

<!-- more -->

Either everything happens, or nothing does—atomicity at work. Working with a monolith is straightforward; you don't have to concern yourself with the complexities of distributed transactions that often arise in microservices architectures. In a monolithic system, everything is contained within a single codebase, making it easier to manage transactions and ensure consistency across the application. 

!!! info "Note"

    It does not mean that a monolith cannot perform dual writes; sometimes, a monolith needs to publish an event to write to another system as well.

## Doing it with Microservices

Let's consider a scenario with two microservices: **order-service** and **report-service**. The **order-service** receives an HTTP request, saves the order, and it's necessary to notify the **report-service** when the order is created/updated. We will see, with code samples, some mistakes that occur when we try to achieve this goal.

### Code 1

```java linenums="1"
public CreateOrderOutput execute(CreateOrderInput input) {
    
    Order order = input.createOrder();
    
    publisher.send(new OrderCreatedEvent(order));

    QuarkusTransaction.requiringNew().run(() -> {
        orderRepository.save(order);
    });

    return new CreateOrderOutput(order);
}
```

It's incredible! We are utilizing a queue, using an event-driven architecture woooow!

**What is the problem here? tic! tac! :clock:**

**Answer:** We are publishing to the queue first, if the database fails the **report-service** will get a incosistent data.

### Code 2

Alright, now I do everything in a single transaction: first, I try to save it in the database, then I send it to the queue. If the queue fails, I don't save it in the database.

```java linenums="1"
public CreateOrderOutput execute(CreateOrderInput input) {
    
    Order order = input.createOrder();

    QuarkusTransaction.requiringNew().run(() -> {
        orderRepository.save(order);
        publisher.send(new OrderCreatedEvent(order));
    });

    return new CreateOrderOutput(order);
}
```

**What is the problem here? tic! tac! tic! tac! :clock: :clock:**

**Answer:** We are adding I/O operation into a database transaction, it is a wrong decision and bad practice.

??? tip "Do not execute I/O operation into the transaction"
        
    The problem with putting I/O operations inside a transaction is that it can cause locking and increase waiting time since transactions typically lock resources until they are completed. Additionally, some I/O operations may not be transactional by nature, which can lead to unexpected behaviors or partially completed transactions in case of failure. Instead, it is generally preferable to perform I/O operations outside the transaction or in a separate transaction, depending on the specific requirements of the system.


### Code 3

Ok, I will try again...

```java linenums="1"
public CreateOrderOutput execute(CreateOrderInput input) {

    Order order = input.createOrder();

    QuarkusTransaction.requiringNew().run(() -> {
        orderRepository.save(order);
    });

    publisher.send(new OrderCreatedEvent(order));

    return new CreateOrderOutput(order);
}
```

**What is the problem here? tic! tac! tic! tac! tic! tac! :clock: :clock: :clock:**

**Answer**: The problem here is that the publishing of the event can fail. If it fails, the **report-service** will not be notified.

## Solving with Outbox Pattern

When we perform dual writes, such as saving order data and sending a message to the queue, it's **challenging to maintain consistency**.

In our situation, we are trying to **save order data** and **send a message** at the same time. To solve this, we need another task that **keeps trying until both actions are done**.

This other task, which will keep retrying until both actions are done, can be implemented using the **Outbox pattern**. The Outbox pattern can be composed with two operations:

1. Save both the order and the message in the database. If the message is sent successfully to the queue, delete it from the database. If not, move to **step 2**.
2. Retry sending the message to the queue. We can do this thanks **step 1**.

Show me the code!

```java linenums="1"
public CreateOrderOutput execute(CreateOrderInput input) {
    
    Order order = input.createOrder();

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
    for (Outbox outbox : outboxs) { // (2)
        this.publisher.send(outbox.getMessage());
        QuarkusTransaction.requiringNew().run(() -> { // (3)
            this.outboxRepository.delete(outbox.getId());
        });
    }
}
```


1. We are using Quarkus `scheduler` for scheduling periodic tasks.
2. For each `Outbox` entry, we are sending it to the queue and storing the result.
3. We are deleting all `Outbox` entries that have been sent within a single transaction.

??? info 

    Adapt the code to fit your needs. Running it `every 5 seconds` is ideal for testing the Outbox retry functionality.

!!! danger "Important"
    Note that our consumer (**report-service**) needs to be idempotent because there's a chance that the **order-service** might send the same message multiple times. Another crucial point to remember is that if I try to send an **OrderCreatedEvent** message and encounter an error, and then shortly after, the status of my order changes to **CANCELED**, there's a possibility that I might send the **CANCELED** event before the **order created event**.

## Considerations

In this post, you've seen how to implement the Outbox pattern. Another method for addressing the dual write issue is by utilizing Change Data Capture (CDC). You can read more about CDC [here](https://en.wikipedia.org/wiki/Change_data_capture).

## Source code

I used the following technologies in the repository:

- Panache
- PostgreSQL
- DevServices
- Scheduler

If you'd like to view the entire code, you can access it [here](https://github.com/mcruzdev/quarkus-outbox-pattern).


## Thank you

That's all; thank you for reading! See you in the next post. Goodbye! 👋
