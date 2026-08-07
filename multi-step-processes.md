# Multi-Step Processes — System Design Pattern

## Table of Contents

1. [The Problem](#the-problem)
2. [Why Distributed Multi-Step Processes Are Hard](#why-distributed-multi-step-processes-are-hard)
3. [Solutions](#solutions)
   - [Single Server Orchestration](#1-single-server-orchestration)
   - [Event Sourcing](#2-event-sourcing)
   - [Workflow Systems](#3-workflow-systems)
   - [Durable Execution Engines](#4-durable-execution-engines)
   - [Managed Workflow Systems](#5-managed-workflow-systems)
4. [Saga Pattern](#saga-pattern)
   - [Choreography](#choreography-saga)
   - [Orchestration](#orchestration-saga)
   - [Compensating Transactions](#compensating-transactions)
5. [Deep Dives](#deep-dives)
   - [Workflow Versioning](#workflow-versioning)
   - [Exactly-Once Execution](#exactly-once-execution)
   - [External Events](#external-events)
   - [State Size Management](#state-size-management)
6. [Real-World Examples](#real-world-examples)
7. [Interview Cheat Sheet](#interview-cheat-sheet)

---

## The Problem

Real production systems must coordinate dozens of services to fulfill a single user request. Consider an **e-commerce order fulfillment**:

```text
1. Charge payment (Stripe)
2. Reserve inventory (Warehouse Service)
3. Create shipping label (FedEx API)
4. Wait for human to pick item (minutes to hours)
5. Send confirmation email (SES)
6. Wait for pickup (hours to days)
```

Each step can:

- **Fail** — external API is down
- **Timeout** — human hasn't picked item for 2 hours
- **Partially succeed** — payment charged but inventory unavailable
- **Require rollback** — if step 3 fails, must refund step 1

And the server orchestrating all this can **crash mid-process**.

```text
                    ┌──────────┐
                    │ Charge   │──FAIL──▶ refund?
                    │ Payment  │
                    └────┬─────┘
                         │ OK
                    ┌────▼─────┐
                    │ Reserve  │──FAIL──▶ refund payment?
                    │Inventory │
                    └────┬─────┘
                         │ OK
                    ┌────▼─────┐
                    │ Shipping │──FAIL──▶ refund + release inventory?
                    │  Label   │
                    └────┬─────┘
                         │ OK
                    ┌────▼─────────────────────────┐
                    │ Wait for human pick (hours)  │──TIMEOUT──▶ ???
                    └──────────────────────────────┘

Server crashes here ──▶ entire state lost
```

---

## Why Distributed Multi-Step Processes Are Hard

### The Six Failure Modes

Even a simple 3-step sequence has hidden failure modes:

```java
// Looks simple. Is not.
chargePayment(order);       // 1. succeeds
reserveInventory(order);    // 2. throws exception — payment already charged!
sendConfirmation(order);    // 3. never reached
```

Problems:

1. **Partial failure** — some steps succeed, others fail
2. **No atomicity** — unlike DB transactions, distributed calls can't roll back atomically
3. **Idempotency** — retrying step 1 may double-charge the customer
4. **State loss** — server restart loses in-memory workflow state
5. **Long waits** — can't hold a thread for hours waiting on a human
6. **Ordering** — events from async systems arrive out of order

---

## Solutions

### 1. Single Server Orchestration

Run all steps sequentially in one process, store state in DB after each step.

```java
@Service
public class OrderOrchestrator {

    @Transactional
    public void processOrder(String orderId) {
        Order order = orderRepo.findById(orderId);

        if (order.getStatus() == PENDING) {
            paymentService.charge(order);
            order.setStatus(PAYMENT_DONE);
            orderRepo.save(order); // checkpoint
        }

        if (order.getStatus() == PAYMENT_DONE) {
            inventoryService.reserve(order);
            order.setStatus(INVENTORY_RESERVED);
            orderRepo.save(order); // checkpoint
        }

        if (order.getStatus() == INVENTORY_RESERVED) {
            shippingService.createLabel(order);
            order.setStatus(LABEL_CREATED);
            orderRepo.save(order); // checkpoint
        }
    }
}
```

**Pros:** Simple, easy to reason about

**Cons:**

- Single point of failure
- Doesn't handle long waits (hours/days)
- Retry logic mixed with business logic
- Doesn't scale horizontally

**When to use:** Very simple flows, low scale, steps complete in seconds

---

### 2. Event Sourcing

Instead of storing current state, store **every event that led to the current state**. Replay events to reconstruct state.

```text
Events stored in append-only log:
  ORDER_CREATED       { orderId, items, timestamp }
  PAYMENT_CHARGED     { orderId, amount, txnId }
  INVENTORY_RESERVED  { orderId, warehouseId }
  LABEL_CREATED       { orderId, trackingNumber }
  ORDER_SHIPPED       { orderId, carrier }
```

Current state = replay all events from beginning.

```java
@Service
public class OrderEventStore {

    public OrderState reconstruct(String orderId) {
        List<OrderEvent> events = eventStore.findByOrderId(orderId);
        OrderState state = new OrderState();
        for (OrderEvent e : events) {
            state.apply(e); // fold events into state
        }
        return state;
    }
}
```

**Pros:**

- Full audit trail
- Can replay to fix bugs
- Decoupled services react to events

**Cons:**

- Complex to query current state (need projections/read models)
- Event schema evolution is hard
- Ordering guarantees needed across services

---

### 3. Workflow Systems

A **workflow** is a durable, resumable definition of a multi-step process. The workflow engine persists state between steps so crashes don't lose progress.

```text
Workflow = code that defines steps + engine that runs it durably
```

Key properties:

- **Durable** — survives server restarts
- **Resumable** — picks up from last completed step
- **Retryable** — automatic retries with backoff
- **Observable** — can query what step a workflow is on

---

### 4. Durable Execution Engines

Durable execution lets you write **normal sequential code** that the engine makes crash-safe by persisting every step's result.

#### Temporal (most common in interviews)

```java
// Workflow definition — looks like normal Java code
@WorkflowInterface
public interface OrderFulfillmentWorkflow {
    @WorkflowMethod
    void fulfillOrder(String orderId);
}

@WorkflowImpl
public class OrderFulfillmentWorkflowImpl implements OrderFulfillmentWorkflow {

    // Activity stubs — each call is durable (retried on failure, result persisted)
    private final PaymentActivities payment = Workflow.newActivityStub(
            PaymentActivities.class,
            ActivityOptions.newBuilder()
                    .setStartToCloseTimeout(Duration.ofSeconds(30))
                    .setRetryOptions(RetryOptions.newBuilder()
                            .setMaximumAttempts(3)
                            .build())
                    .build());

    private final InventoryActivities inventory = Workflow.newActivityStub(InventoryActivities.class, ...);
    private final ShippingActivities shipping = Workflow.newActivityStub(ShippingActivities.class, ...);
    private final NotificationActivities notification = Workflow.newActivityStub(NotificationActivities.class, ...);

    @Override
    public void fulfillOrder(String orderId) {
        // Each line is durable — if server crashes here, Temporal replays from last checkpoint
        payment.charge(orderId);
        inventory.reserve(orderId);
        String trackingNumber = shipping.createLabel(orderId);

        // Wait for human action — can sleep for hours/days without holding a thread
        Workflow.sleep(Duration.ofHours(2)); // durable timer, not Thread.sleep

        // Wait for external signal (e.g., item picked by warehouse worker)
        Workflow.await(() -> itemPicked); // blocks workflow, not a thread

        notification.sendConfirmation(orderId, trackingNumber);
    }

    // External signal handler — warehouse worker scans item
    @SignalMethod
    public void itemPickedUp() {
        this.itemPicked = true;
    }

    // Query handler — check what step the order is on
    @QueryMethod
    public String getStatus() {
        return currentStep;
    }
}
```

**How Temporal makes this durable:**

```text
Workflow code runs → Temporal records each activity result in its DB
Server crashes mid-workflow → Temporal replays history to restore state
Activity fails → Temporal retries with configured backoff automatically
Workflow sleeps for 2h → timer stored in Temporal, no thread held
```

#### Activity implementations (the actual business logic)

```java
@ActivityImpl
public class PaymentActivitiesImpl implements PaymentActivities {

    @Override
    public void charge(String orderId) {
        Order order = orderRepo.findById(orderId);
        // must be idempotent — Temporal may retry this
        if (!order.isPaymentCharged()) {
            stripeClient.charge(order.getAmount(), order.getIdempotencyKey());
            order.setPaymentCharged(true);
            orderRepo.save(order);
        }
    }
}
```

---

### 5. Managed Workflow Systems

Alternatives to Temporal for specific use cases:

| System | Type | Best For |
|--------|------|----------|
| **Temporal** | Durable execution | General-purpose, complex business logic |
| **AWS Step Functions** | State machine (JSON) | AWS-native, simple state machines |
| **Apache Airflow** | DAG scheduler | Data pipelines, batch jobs |
| **Conductor (Netflix)** | Microservice orchestration | Microservice workflows |
| **Camunda** | BPMN engine | Human-in-the-loop, compliance workflows |
| **AWS SWF** | Legacy workflow | Simple workflows on AWS |

---

## Saga Pattern

A **Saga** is a sequence of local transactions where each step publishes an event or message. If a step fails, compensating transactions undo previous steps.

### Choreography Saga

Services react to each other's events — no central coordinator.

```text
Order Service ──ORDER_CREATED──▶ Payment Service
                                        │
                                 PAYMENT_CHARGED
                                        │
                                        ▼
                               Inventory Service
                                        │
                                 INVENTORY_RESERVED
                                        │
                                        ▼
                               Shipping Service
```

**On failure (Inventory unavailable):**

```text
Inventory Service ──INVENTORY_FAILED──▶ Payment Service
                                               │
                                        compensate: refund payment
```

```java
// Inventory Service listens for payment event
@KafkaListener(topics = "payment-charged")
public void onPaymentCharged(PaymentChargedEvent event) {
    try {
        inventoryService.reserve(event.getOrderId());
        kafkaTemplate.send("inventory-reserved", new InventoryReservedEvent(event.getOrderId()));
    } catch (InsufficientInventoryException e) {
        // trigger compensation
        kafkaTemplate.send("inventory-failed", new InventoryFailedEvent(event.getOrderId()));
    }
}

// Payment Service listens for inventory failure → compensate
@KafkaListener(topics = "inventory-failed")
public void onInventoryFailed(InventoryFailedEvent event) {
    paymentService.refund(event.getOrderId()); // compensating transaction
}
```

**Pros:** Loose coupling, no single point of failure

**Cons:** Hard to debug, no central view of saga state, complex error handling

---

### Orchestration Saga

A central **orchestrator** tells each service what to do and handles failures explicitly.

```text
                    ┌─────────────────┐
                    │  Saga           │
                    │  Orchestrator   │
                    └──┬──────────┬───┘
                       │          │
              charge   │          │  reserve
                       ▼          ▼
              Payment Svc    Inventory Svc
```

```java
@Service
public class OrderSagaOrchestrator {

    public void execute(String orderId) {
        try {
            paymentService.charge(orderId);
        } catch (Exception e) {
            // nothing to compensate yet
            throw new SagaException("Payment failed", e);
        }

        try {
            inventoryService.reserve(orderId);
        } catch (Exception e) {
            // compensate: refund payment
            paymentService.refund(orderId);
            throw new SagaException("Inventory failed, payment refunded", e);
        }

        try {
            shippingService.createLabel(orderId);
        } catch (Exception e) {
            // compensate in reverse order
            inventoryService.release(orderId);
            paymentService.refund(orderId);
            throw new SagaException("Shipping failed, saga rolled back", e);
        }
    }
}
```

**Pros:** Central visibility, clear compensation logic, easier to debug

**Cons:** Orchestrator becomes a bottleneck and point of failure (use Temporal to make it durable)

---

### Compensating Transactions

Each step must have a compensating action that **undoes its effect**:

| Step | Compensating Action |
|------|---------------------|
| Charge payment | Refund payment |
| Reserve inventory | Release reservation |
| Create shipping label | Cancel label |
| Send email | Cannot undo — make idempotent instead |
| Create DB record | Delete record |

**Important:** Compensations must also be **idempotent** — they may be retried too.

```java
@ActivityImpl
public class PaymentActivitiesImpl implements PaymentActivities {

    @Override
    public void refund(String orderId) {
        // idempotent: check before refunding
        if (!orderRepo.isRefunded(orderId)) {
            stripeClient.refund(orderId);
            orderRepo.markRefunded(orderId);
        }
    }
}
```

---

## Deep Dives

### Workflow Versioning

**Problem:** You have 10,000 active order workflows. You need to add a new step (fraud check) between payment and inventory. How do you update running workflows?

**Never change the code a running workflow depends on** — Temporal replays history using the original code.

```java
@Override
public void fulfillOrder(String orderId) {
    payment.charge(orderId);

    // Version gate — old workflows skip this block, new ones execute it
    int version = Workflow.getVersion("fraud-check", Workflow.DEFAULT_VERSION, 1);
    if (version >= 1) {
        fraudCheck.verify(orderId); // new step, only for workflows started after this deploy
    }

    inventory.reserve(orderId);
    shipping.createLabel(orderId);
    notification.sendConfirmation(orderId);
}
```

**Workflow Migration Strategy:**

1. Deploy new code with version gates
2. Old workflows complete using old path
3. New workflows use new path
4. Once all old workflows complete, remove version gate in next deploy

---

### Exactly-Once Execution

**Problem:** A payment activity fails after charging but before returning success. Temporal retries → double charge.

**Solution: Idempotency keys**

```java
@ActivityImpl
public class PaymentActivitiesImpl implements PaymentActivities {

    @Override
    public void charge(String orderId) {
        // idempotency key = orderId (stable across retries)
        String idempotencyKey = "charge-" + orderId;

        // Stripe deduplicates using idempotency key
        stripeClient.charge(
            ChargeRequest.builder()
                .amount(getAmount(orderId))
                .idempotencyKey(idempotencyKey) // same key = same result returned
                .build()
        );
    }
}
```

**At-least-once + idempotency = effectively exactly-once**

For internal DB operations:

```java
// Upsert instead of insert — safe to call multiple times
orderRepo.upsert(orderId, status); // INSERT ... ON CONFLICT DO UPDATE
```

---

### External Events

**Problem:** Workflow needs to wait for a human action (warehouse worker scans item) or an external webhook (payment gateway callback).

**Solution: Signals in Temporal**

```java
// Workflow waits for signal
@WorkflowImpl
public class OrderWorkflowImpl implements OrderWorkflow {

    private boolean itemScanned = false;

    @Override
    public void fulfillOrder(String orderId) {
        payment.charge(orderId);
        inventory.reserve(orderId);
        shipping.createLabel(orderId);

        // Durable wait — no thread held, survives server restart
        // Timeout after 48 hours if no signal received
        boolean received = Workflow.await(Duration.ofHours(48), () -> itemScanned);
        if (!received) {
            // timeout — cancel order
            handleTimeout(orderId);
            return;
        }

        notification.sendConfirmation(orderId);
    }

    @SignalMethod
    public void itemScannedByWorker() {
        this.itemScanned = true;
    }
}

// REST endpoint — warehouse worker app calls this when item is scanned
@RestController
public class WarehouseController {

    private final WorkflowClient workflowClient;

    @PostMapping("/warehouse/scan/{orderId}")
    public void scanItem(@PathVariable String orderId) {
        OrderWorkflow workflow = workflowClient.newWorkflowStub(
                OrderWorkflow.class,
                WorkflowOptions.newBuilder().setWorkflowId(orderId).build());

        workflow.itemScannedByWorker(); // sends signal to running workflow
    }
}
```

**For webhooks (e.g., payment gateway callback):**

```java
@PostMapping("/webhooks/payment")
public void paymentWebhook(@RequestBody PaymentEvent event) {
    OrderWorkflow workflow = workflowClient.newWorkflowStub(
            OrderWorkflow.class, event.getOrderId());
    workflow.paymentConfirmed(event.getStatus()); // signal the workflow
}
```

---

### State Size Management

**Problem:** Workflow accumulates large state over time. Temporal stores all this in its history → history grows large → replay is slow.

**Rules:**

1. **Store only IDs in workflow state, not full objects**
2. **Fetch data from DB inside activities** (activities don't contribute to workflow history size)
3. **Use Continue-as-New** for very long-running workflows

```java
// BAD — stores large object in workflow state
private OrderDetails orderDetails; // entire order with all items, images, etc.

// GOOD — store only the ID, fetch when needed inside activities
private String orderId; // just the key

@ActivityImpl
public class ShippingActivitiesImpl implements ShippingActivities {
    @Override
    public String createLabel(String orderId) {
        Order order = orderRepo.findById(orderId); // fetch full data inside activity
        return fedex.createLabel(order);
    }
}
```

**Continue-as-New** — for workflows that run for months with thousands of events:

```java
// When history gets too large, start fresh workflow with current state
if (Workflow.getInfo().getHistoryLength() > 10_000) {
    Workflow.continueAsNew(orderId); // new workflow, same ID, clean history
}
```

---

## Real-World Examples

### Uber — Ride Fulfillment

```text
Trip Request Workflow:
  1. Match driver (retry until found, timeout after 5 min)
  2. Wait for driver acceptance (signal)
  3. Track driver location (periodic activity)
  4. Wait for trip start (signal)
  5. Wait for trip end (signal)
  6. Charge payment
  7. Send receipt

Compensations:
  Driver cancels → re-match (loop back to step 1)
  Payment fails → retry 3x → mark debt, notify user
```

- **Engine:** Cadence (Uber's open-source predecessor to Temporal)
- **External events:** Driver app sends signals (accept, start, end trip)
- **Long waits:** `Workflow.await()` for each driver action

---

### Payment System — Stripe-like

```text
Payment Processing Workflow:
  1. Validate card
  2. Send to card network (Visa/Mastercard) — external, async
  3. Wait for network response (webhook signal, timeout 30s)
  4. If approved: settle funds
  5. If declined: notify user
  6. If timeout: query network status (activity), retry or void

Compensations:
  Settlement fails → void authorization
  Duplicate charge detected → refund via idempotency key
```

- **Idempotency:** Every payment has a stable idempotency key
- **Exactly-once:** Card network deduplicates using idempotency key
- **External event:** Card network webhook signals the workflow

---

### E-Commerce Order Fulfillment

```text
Order Workflow (Temporal):

  payment.charge(orderId)                    // Activity, retries 3x
       │
  inventory.reserve(orderId)                 // Activity, retries 3x
       │                                     // On fail: payment.refund()
  shipping.createLabel(orderId)              // Activity, retries 3x
       │                                     // On fail: inventory.release() + payment.refund()
  Workflow.await(48h, () -> itemScanned)     // Durable wait for human
       │
  notification.sendConfirmation(orderId)     // Activity
       │
  Workflow.await(7d, () -> delivered)        // Wait for delivery
       │
  payment.settle(orderId)                    // Final settlement
```

---

## Interview Cheat Sheet

### When to Use Which Solution

| Scenario | Solution |
|----------|----------|
| Simple 2-3 steps, fast, low scale | Single server orchestration |
| Need full audit trail, event-driven | Event Sourcing + Kafka |
| Complex multi-step, crash-safe | Temporal (durable execution) |
| Microservice choreography, loose coupling | Choreography Saga |
| Central control, clear rollback | Orchestration Saga |
| AWS-native, simple state machine | AWS Step Functions |
| Data pipelines, batch jobs | Apache Airflow |
| Human-in-the-loop, compliance | Camunda (BPMN) |

---

### Saga: Choreography vs Orchestration

| Aspect | Choreography | Orchestration |
|--------|--------------|---------------|
| Coordination | Events (Kafka) | Central orchestrator |
| Coupling | Loose | Tighter |
| Visibility | Hard (distributed) | Easy (single place) |
| Failure handling | Each service handles own | Orchestrator handles all |
| Debug complexity | High | Low |
| SPOF | No | Yes (mitigate with Temporal) |
| Best for | Simple, stable flows | Complex, changing flows |

---

### Common Deep Dive Answers

**"How do you handle updates to the workflow?"**

- Use `Workflow.getVersion()` to gate new steps
- Old running workflows skip new steps, new workflows execute them
- Remove version gates once all old workflows complete

**"How do you ensure exactly-once execution?"**

- Idempotency keys on all external calls (Stripe, etc.)
- Upsert (not insert) for DB operations
- Check-before-act pattern in all activities

**"How do you handle external events (webhooks, human actions)?"**

- Expose a REST endpoint that sends a Temporal signal to the running workflow
- `Workflow.await(timeout, condition)` for durable waits without holding threads

**"How do you keep workflow state size in check?"**

- Store only IDs in workflow state, fetch full objects inside activities
- Use `continueAsNew` for very long-running workflows
- Set history size limits and alert on them

**"How do you handle partial failures?"**

- Each step has a compensating action (refund, release, cancel)
- Compensations must be idempotent
- Execute compensations in reverse order on failure

---

### Architecture Decision Flowchart

```text
Multi-step process needed?
          │
          ▼
Steps complete in seconds and server is single instance?
   YES → Single server orchestration (checkpointing in DB)
   NO  → continue
          │
          ▼
Need full audit trail / event replay?
   YES → Event Sourcing + Kafka
   NO  → continue
          │
          ▼
Steps involve external waits (hours/days) or human actions?
   YES → Temporal (durable execution)
   NO  → continue
          │
          ▼
Services are loosely coupled and flows are stable?
   YES → Choreography Saga (Kafka events)
   NO  → Orchestration Saga (central coordinator)
          │
          ▼
Running on AWS and flow is simple state machine?
   YES → AWS Step Functions
   NO  → Temporal
```

---

### Key Terms Quick Reference

| Term | Meaning |
|------|---------|
| **Saga** | Sequence of local transactions with compensating actions |
| **Compensating transaction** | Action that undoes a previous step |
| **Durable execution** | Code that survives crashes by persisting step results |
| **Idempotency** | Safe to call multiple times with same result |
| **Signal** | External event sent to a running workflow |
| **Activity** | Single step in a Temporal workflow (retried on failure) |
| **Continue-as-New** | Restart workflow with clean history, same ID |
| **Workflow versioning** | Gate new steps behind version checks for safe deploys |
| **Orchestration** | Central coordinator tells services what to do |
| **Choreography** | Services react to each other's events, no coordinator |
