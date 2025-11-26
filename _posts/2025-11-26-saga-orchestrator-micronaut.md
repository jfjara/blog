---
layout: single
title: "Implementing the Saga Pattern (Orchestrated Version) with Micronaut"
excerpt: "In distributed systems development, one of the biggest challenges is maintaining data consistency when an operation spans multiple independent services"
date: 2025-11-26
classes: wide
header:
  teaser: /assets/images/htb-writeup-delivery/delivery_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - java
  - micronaut
  - architecture
  - practise
tags:  
  - java
  - micronaut
  - saga
  - architecture
---

![](/assets/images/htb-writeup-delivery/delivery_logo.png)


A practical and simple example to understand distributed transactions.

In distributed systems development, one of the biggest challenges is maintaining data consistency when an operation spans multiple independent services.~~  
The classic example: a user places an order, payment is processed, and shipment is handled. Three services, three responsibilities, one single flow.

In a monolithic architecture, it’s simple: one big transaction covers everything.  
In microservices… that doesn’t exist.

This is where the Saga pattern comes in.

In this project, I implemented a small, clear, and functional example of the orchestrated Saga pattern using Micronaut.  
It’s not a production system, but a step-by-step educational example you can follow without getting lost.

---

## 🎯 What This Example Solves

The example demonstrates:

- Creating an order.
- Attempting to charge the user.
- Attempting to ship the product.
- Executing compensations if something fails (e.g., refunding the user).
- Ensuring the final order state reflects reality.

Above all, it makes the Saga pattern logic easy to understand at first glance.

No Kafka queues, asynchronous messaging, or sophisticated distributed transactions.  
This is a clean, sequential, conceptual example.

---

## 🧠 Why Orchestrated?

The Saga pattern can be implemented in two ways:

- **Choreographed**  
  Each microservice reacts to events and triggers the next.  
  Great for fully distributed systems, but hard to follow in a simple example.

- **Orchestrated**  
  A central orchestrator controls the flow.  
  Explicit, readable, and educational.

This project uses the **orchestrated approach**.

---

## 🏗️ Example Architecture

The project consists of:

- `order-service` → creates orders and manages their state. In this example, this service contains the orchestrator.
- `payment-service` → simulates payments and refunds
- `shipment-service` → simulates shipments, stock, and failures

Each service is an independent Micronaut microservice.

---

## 🔄 Complete Saga Flow

The core logic resides in the `OrderOrchestrator` inside the orchestrator service.

Here’s the most important part:

```java
public OrderResponseDTO manage(final OrderDTO orderDTO) {
    final String orderId = orderRepository.create(orderDTO);

    try {
        orderRepository.updateStatus(orderId, OrderStatusDTO.PENDING);

        PaymentStatusDTO paymentResult = executePayment(orderDTO);

        if (isPaymentFailed(paymentResult)) {
            return failOrder(orderId, OrderStatusDTO.PAYMENT_FAILED);
        }

        orderRepository.updateStatus(orderId, OrderStatusDTO.PAID);

        ShipmentStatusDTO shipmentResult = executeShipment(orderId, orderDTO);

        if (shipmentResult == ShipmentStatusDTO.OUT_OF_STOCK) {
            refund(orderDTO);
            return failOrder(orderId, OrderStatusDTO.OUT_OF_STOCK);
        }

        if (shipmentResult == ShipmentStatusDTO.FAILED) {
            refund(orderDTO);
            return failOrder(orderId, OrderStatusDTO.CANCELLED);
        }

        orderRepository.updateStatus(orderId, OrderStatusDTO.SHIPPED);
        return new OrderResponseDTO(orderId, OrderStatusDTO.SHIPPED);

    } catch (PaymentException e) {
        return failOrder(orderId, OrderStatusDTO.PAYMENT_FAILED);
    }
}

```

---


## 🧵 Step 1: Create the Order

```java
final String orderId = orderRepository.create(orderDTO);
orderRepository.updateStatus(orderId, OrderStatusDTO.PENDING);
```

We create a new order and set it to PENDING. This marks the start of the Saga.

## 💳 Step 2: Attempt Payment

```java
PaymentStatusDTO paymentResult = executePayment(orderDTO);
```

executePayment delegates to the payment service:

```java
private PaymentStatusDTO executePayment(OrderDTO orderDTO) {
return paymentRepository
.pay(orderDTO.userId(), orderDTO.product().price())
.orElse(PaymentStatusDTO.FAILED);
}
```

Payment can return:

```
OK

WITHOUT_BALANCE

FAILED
```

If the result is a failure:

```java
private boolean isPaymentFailed(PaymentStatusDTO status) {
return status == PaymentStatusDTO.WITHOUT_BALANCE
|| status == PaymentStatusDTO.FAILED;
}
```

The Saga stops:

```java
return failOrder(orderId, OrderStatusDTO.PAYMENT_FAILED);
```

## 📦 Step 3: Attempt Shipment

If payment succeeds:

```java
ShipmentStatusDTO shipmentResult = executeShipment(orderId, orderDTO);

private ShipmentStatusDTO executeShipment(String orderId, OrderDTO orderDTO) {
return shipmentRepository
.call(orderId, orderDTO.product().id())
.orElse(ShipmentStatusDTO.FAILED);
}
```

Possible failures:

```
OUT_OF_STOCK → no stock, need to compensate

FAILED → shipment error, also need to compensate
```

In both cases:

```java
refund(orderDTO);
return failOrder(orderId, OrderStatusDTO.OUT_OF_STOCK); // or CANCELLED
```

## 💸 Compensation Mechanism (Refund)

```java
private void refund(OrderDTO orderDTO) {
paymentRepository.refund(orderDTO.userId(), orderDTO.product().price());
}
```

Compensation is the essence of the Saga: undo previous steps if something goes wrong.

## 🚚 Step 4: Success → SHIPPED

If everything goes well:

```java
orderRepository.updateStatus(orderId, OrderStatusDTO.SHIPPED);
return new OrderResponseDTO(orderId, OrderStatusDTO.SHIPPED);
```

Saga completed.

## 🗺️ What This Example Shows

This project:

- A realistic (but simplified) Saga flow
- Clearly shows where compensations occur
- Separates responsibilities across microservices
- Uses Micronaut in a light and clean way
- Keeps orchestrator logic easy to follow

It’s perfect for anyone learning the Saga pattern from scratch.

## 📌 What It Is Not

Not a production-ready system. Intentionally, it does not include:

- Idempotency
- Retries
- Timeouts
- Asynchronous messaging
- True distributed consistency
- Persistent storage
- Advanced resilience

This is deliberate to keep the example:

```
 👉 small, clear, and educational.
 ```

## 🧩 Conclusion

The Saga pattern is essential for distributed architectures, enabling complex transactions without locking systems or relying on global transactions.

This project demonstrates a simple, readable, and functional orchestrated Saga implementation using Micronaut.
It’s an ideal starting point to study, modify, or experiment with different variations.