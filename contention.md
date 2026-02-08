# Contention in Distributed Systems

> **Reference**: [The Pragmatic Engineer - Contention](https://newsletter.pragmaticengineer.com/p/contention)

## Table of Contents
1. [What is Contention?](#what-is-contention)
2. [Single Node Solutions](#single-node-solutions)
3. [Distributed Solutions](#distributed-solutions)
4. [Real-world Examples](#real-world-examples)
5. [Trade-offs and When to Use](#trade-offs-and-when-to-use)

---

## What is Contention?

**Contention** occurs when multiple processes or threads compete for the same resource simultaneously. In distributed systems, this becomes particularly challenging when:
- Multiple users try to book the same concert ticket
- Concurrent transactions attempt to update the same bank account
- Multiple services compete to update shared inventory

### The Core Problem

```
Time →
User A: Read balance ($100) → Calculate ($100-$50) → Write ($50)
User B:        Read balance ($100) → Calculate ($100-$30) → Write ($70)

Result: Final balance = $70 (Lost update! Should be $20)
```

### Types of Contention Issues

1. **Lost Updates**: One transaction overwrites another's changes
2. **Dirty Reads**: Reading uncommitted data from another transaction
3. **Non-repeatable Reads**: Same query returns different results within a transaction
4. **Phantom Reads**: New rows appear in subsequent queries

---

## Single Node Solutions

### 1. Atomicity with Database Transactions

**ACID Properties** ensure data consistency:
- **Atomicity**: All or nothing execution
- **Consistency**: Data remains valid
- **Isolation**: Transactions don't interfere
- **Durability**: Committed data persists

```sql
BEGIN TRANSACTION;

UPDATE accounts 
SET balance = balance - 50 
WHERE account_id = 123 AND balance >= 50;

-- Check if update succeeded
IF @@ROWCOUNT = 0
    ROLLBACK;
ELSE
    COMMIT;
```

**Pros**: Simple, built-in database support  
**Cons**: Blocks other transactions, can cause deadlocks

---

### 2. Pessimistic Locking

**Lock the resource** before accessing it, preventing others from modifying until released.

#### Row-Level Locking
```sql
BEGIN TRANSACTION;

-- Lock the row for update
SELECT balance FROM accounts 
WHERE account_id = 123 
FOR UPDATE;

-- Perform operations
UPDATE accounts 
SET balance = balance - 50 
WHERE account_id = 123;

COMMIT;
```

#### Application-Level Locking
```java
import java.util.concurrent.locks.ReentrantLock;

public class AccountService {
    private final ReentrantLock lock = new ReentrantLock();
    
    public boolean withdraw(String accountId, double amount) {
        lock.lock();
        try {
            double balance = getBalance(accountId);
            if (balance >= amount) {
                double newBalance = balance - amount;
                updateBalance(accountId, newBalance);
                return true;
            }
            return false;
        } finally {
            lock.unlock();
        }
    }
}
```

**Pros**: 
- Prevents conflicts entirely
- Simple to reason about

**Cons**: 
- Reduced throughput (serializes access)
- Risk of deadlocks
- Can cause contention bottlenecks

---

### 3. Optimistic Concurrency Control (OCC)

**Assume conflicts are rare**. Check before committing if data has changed.

#### Version-Based OCC
```sql
-- Read with version
SELECT balance, version FROM accounts WHERE account_id = 123;
-- Returns: balance=100, version=5

-- Update only if version hasn't changed
UPDATE accounts 
SET balance = 50, version = 6
WHERE account_id = 123 AND version = 5;

-- Check rows affected
IF @@ROWCOUNT = 0
    -- Retry: version changed, conflict detected
```

#### Timestamp-Based OCC
```java
import java.time.Instant;
import java.util.concurrent.TimeUnit;

public class Account {
    private double balance;
    private Instant lastModified;
    
    public Account() {
        this.balance = 100.0;
        this.lastModified = Instant.now();
    }
    
    // Getters and setters
}

public class OptimisticAccountService {
    private static final int MAX_RETRIES = 3;
    
    public boolean withdrawOptimistic(String accountId, double amount) {
        for (int attempt = 0; attempt < MAX_RETRIES; attempt++) {
            // Read current state
            Account account = readAccount(accountId);
            Instant originalTimestamp = account.getLastModified();
            
            if (account.getBalance() < amount) {
                return false;
            }
            
            // Try to update with timestamp check
            boolean success = updateAccount(
                accountId,
                account.getBalance() - amount,
                Instant.now(),
                originalTimestamp
            );
            
            if (success) {
                return true;
            }
            
            // Conflict detected, retry with backoff
            try {
                TimeUnit.MILLISECONDS.sleep((long) (100 * Math.pow(2, attempt)));
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return false;
            }
        }
        
        return false; // Max retries exceeded
    }
}
```

**Pros**: 
- Higher throughput (no locks)
- No deadlocks
- Good for read-heavy workloads

**Cons**: 
- Retry logic required
- Wasted work on conflicts
- Poor performance under high contention

---

### 4. Isolation Levels

Different isolation levels offer trade-offs between consistency and performance:

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|----------------|------------|---------------------|--------------|-------------|
| Read Uncommitted | Yes | Yes | Yes | Highest |
| Read Committed | No | Yes | Yes | High |
| Repeatable Read | No | No | Yes | Medium |
| Serializable | No | No | No | Lowest |

```sql
-- Set isolation level
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

BEGIN TRANSACTION;
-- Your queries here
COMMIT;
```

**Example: Read Committed**
```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;

public class DatabaseService {
    public void queryWithIsolation(String productId) throws Exception {
        Connection conn = DriverManager.getConnection(dbUrl, username, password);
        
        // Set isolation level to READ COMMITTED
        conn.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED);
        
        PreparedStatement stmt = conn.prepareStatement(
            "SELECT * FROM inventory WHERE product_id = ?"
        );
        stmt.setString(1, productId);
        
        ResultSet rs = stmt.executeQuery();
        // Only sees committed data
        
        rs.close();
        stmt.close();
        conn.close();
    }
}
```

---

## Distributed Solutions

### 1. Two-Phase Commit (2PC)

**Coordinates transactions** across multiple nodes with a prepare phase and commit phase.

```
Coordinator                 Node A              Node B
    |                         |                   |
    |---Prepare-------------->|                   |
    |---Prepare----------------------------->|
    |                         |                   |
    |<--Vote: Yes-------------|                   |
    |<--Vote: Yes---------------------------|
    |                         |                   |
    |---Commit--------------->|                   |
    |---Commit----------------------------->|
    |                         |                   |
    |<--Ack-------------------|                   |
    |<--Ack------------------------------|
```

```java
import java.util.*;

public class TwoPhaseCommitCoordinator {
    
    public boolean executeTransaction(List<Participant> participants, 
                                     List<Operation> operations) {
        String transactionId = UUID.randomUUID().toString();
        
        // Phase 1: Prepare
        List<String> votes = new ArrayList<>();
        for (Participant participant : participants) {
            String vote = participant.prepare(transactionId, operations);
            votes.add(vote);
            
            if ("NO".equals(vote)) {
                // Abort all
                sendAbort(participants, transactionId);
                return false;
            }
        }
        
        // Phase 2: Commit
        boolean allYes = votes.stream().allMatch(v -> "YES".equals(v));
        if (allYes) {
            sendCommit(participants, transactionId);
            return true;
        } else {
            sendAbort(participants, transactionId);
            return false;
        }
    }
    
    private void sendAbort(List<Participant> participants, String txId) {
        participants.forEach(p -> p.abort(txId));
    }
    
    private void sendCommit(List<Participant> participants, String txId) {
        participants.forEach(p -> p.commit(txId));
    }
}
```

**Pros**: 
- Strong consistency
- Atomic across multiple systems

**Cons**: 
- Blocking protocol (locks held during prepare)
- Single point of failure (coordinator)
- High latency
- Not suitable for high-scale systems

---

### 2. Distributed Locks

**Centralized locking** across multiple nodes using coordination services.

#### Redis-based Distributed Lock (Redlock)
```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.params.SetParams;
import java.util.*;

public class RedisLock {
    private final List<Jedis> redisClients;
    private final String resource;
    private final int ttl; // milliseconds
    private final String token;
    
    public RedisLock(List<Jedis> redisClients, String resource, int ttl) {
        this.redisClients = redisClients;
        this.resource = resource;
        this.ttl = ttl;
        this.token = UUID.randomUUID().toString();
    }
    
    public boolean acquire() {
        long startTime = System.currentTimeMillis();
        int acquired = 0;
        
        // Try to acquire lock on majority of nodes
        for (Jedis client : redisClients) {
            if (acquireSingle(client)) {
                acquired++;
            }
        }
        
        long elapsed = System.currentTimeMillis() - startTime;
        
        // Check if we acquired majority and within time limit
        int majority = redisClients.size() / 2 + 1;
        if (acquired >= majority && elapsed < ttl) {
            return true;
        } else {
            release();
            return false;
        }
    }
    
    private boolean acquireSingle(Jedis client) {
        SetParams params = new SetParams()
            .nx()           // Only set if not exists
            .px(ttl);       // Expiry in milliseconds
        
        String result = client.set(resource, token, params);
        return "OK".equals(result);
    }
    
    public void release() {
        // Lua script for atomic check-and-delete
        String script = 
            "if redis.call('get', KEYS[1]) == ARGV[1] then " +
            "    return redis.call('del', KEYS[1]) " +
            "else " +
            "    return 0 " +
            "end";
        
        for (Jedis client : redisClients) {
            client.eval(script, 
                Collections.singletonList(resource),
                Collections.singletonList(token));
        }
    }
}
```

#### ZooKeeper-based Lock
```java
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.locks.InterProcessMutex;
import org.apache.curator.retry.ExponentialBackoffRetry;

public class ZooKeeperLockExample {
    
    public void executeWithLock() throws Exception {
        CuratorFramework client = CuratorFrameworkFactory.newClient(
            "127.0.0.1:2181",
            new ExponentialBackoffRetry(1000, 3)
        );
        client.start();
        
        InterProcessMutex lock = new InterProcessMutex(client, "/locks/resource-123");
        
        try {
            // Acquire the lock
            lock.acquire();
            
            // Critical section
            // Only one process across cluster can execute this
            processPayment();
            
        } finally {
            // Always release the lock
            lock.release();
            client.close();
        }
    }
    
    private void processPayment() {
        // Payment processing logic
    }
}
```

**Pros**: 
- Works across multiple nodes
- Prevents concurrent access

**Cons**: 
- Additional infrastructure (Redis, ZooKeeper, etcd)
- Network partitions can cause issues
- Lock timeout management complexity

---

### 3. Saga Pattern

**Compensating transactions** instead of global locks for long-running business processes.

#### Choreography-Based Saga
```java
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Service;

// Each service listens to events and reacts

// Order Service
@Service
public class OrderService {
    private final ApplicationEventPublisher eventPublisher;
    
    public OrderService(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }
    
    public void createOrder(Order order) {
        order.setStatus("PENDING");
        saveOrder(order);
        
        // Publish event
        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(),
            order.getUserId(),
            order.getItems()
        );
        eventPublisher.publishEvent(event);
    }
    
    @EventListener
    public void onInventoryFailed(InventoryReservationFailedEvent event) {
        Order order = getOrder(event.getOrderId());
        order.setStatus("CANCELLED");
        saveOrder(order);
        
        // Trigger refund if payment was taken
        OrderCancelledEvent cancelEvent = new OrderCancelledEvent(event.getOrderId());
        eventPublisher.publishEvent(cancelEvent);
    }
}

// Inventory Service
@Service
public class InventoryService {
    private final ApplicationEventPublisher eventPublisher;
    
    public InventoryService(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }
    
    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        boolean success = reserveItems(event.getItems());
        
        if (success) {
            InventoryReservedEvent reservedEvent = 
                new InventoryReservedEvent(event.getOrderId());
            eventPublisher.publishEvent(reservedEvent);
        } else {
            InventoryReservationFailedEvent failedEvent = 
                new InventoryReservationFailedEvent(event.getOrderId());
            eventPublisher.publishEvent(failedEvent);
        }
    }
}
```

#### Orchestration-Based Saga
```java
import org.springframework.stereotype.Service;
import java.util.UUID;

@Service
public class SagaOrchestrator {
    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    private final ShippingService shippingService;
    
    public SagaOrchestrator(InventoryService inventoryService,
                           PaymentService paymentService,
                           ShippingService shippingService) {
        this.inventoryService = inventoryService;
        this.paymentService = paymentService;
        this.shippingService = shippingService;
    }
    
    public boolean executeOrderSaga(Order order) {
        String sagaId = UUID.randomUUID().toString();
        
        try {
            // Step 1: Reserve inventory
            InventoryResult inventoryResult = inventoryService.reserve(order.getItems());
            logStep(sagaId, "inventory", inventoryResult);
            
            try {
                // Step 2: Process payment
                PaymentResult paymentResult = paymentService.charge(order.getTotal());
                logStep(sagaId, "payment", paymentResult);
                
                try {
                    // Step 3: Arrange shipping
                    ShippingResult shippingResult = 
                        shippingService.schedule(order.getAddress());
                    logStep(sagaId, "shipping", shippingResult);
                    
                    return true;
                    
                } catch (ShippingException e) {
                    // Compensate: refund and release inventory
                    paymentService.refund(order.getTotal());
                    inventoryService.release(order.getItems());
                    return false;
                }
                
            } catch (PaymentException e) {
                // Compensate: release inventory
                inventoryService.release(order.getItems());
                return false;
            }
            
        } catch (InventoryException e) {
            // Compensate: nothing to undo yet
            return false;
        }
    }
    
    private void logStep(String sagaId, String step, Object result) {
        // Log saga step execution
    }
}
```

**Pros**: 
- No distributed locks
- Eventual consistency
- Better availability
- Scales well

**Cons**: 
- Complex to implement
- Eventual consistency (not immediate)
- Compensation logic required
- Harder to reason about

---

## Real-world Examples

### Example 1: Concert Ticket Booking

**Problem**: 10,000 users trying to book 100 tickets simultaneously.

**Solution: Optimistic Locking with Queue**

```java
import org.springframework.stereotype.Service;
import java.util.*;

@Service
public class TicketBookingService {
    private final EventRepository eventRepository;
    private final BookingRepository bookingRepository;
    private final BookingQueue queue;
    
    public BookingResponse bookTicket(String eventId, String userId) {
        // Use optimistic locking for inventory check
        Event event = getEventWithVersion(eventId);
        
        if (event.getAvailableTickets() <= 0) {
            return new BookingResponse(false, "sold_out", null);
        }
        
        // Try to decrement atomically
        boolean success = updateEvent(
            eventId,
            event.getAvailableTickets() - 1,
            event.getVersion()
        );
        
        if (!success) {
            // Version mismatch, retry
            return bookTicket(eventId, userId);
        }
        
        // Create booking record
        Booking booking = createBooking(userId, eventId);
        
        // Add to queue for payment (5 min window)
        queue.add(booking.getId(), 300); // TTL in seconds
        
        return new BookingResponse(true, null, booking.getId());
    }
    
    public void releaseUnpaidTickets() {
        // Background job to release expired bookings
        List<String> expired = queue.getExpired();
        for (String bookingId : expired) {
            cancelBooking(bookingId);
            Booking booking = bookingRepository.findById(bookingId).orElseThrow();
            incrementAvailableTickets(booking.getEventId());
        }
    }
}

class BookingResponse {
    private final boolean success;
    private final String reason;
    private final String bookingId;
    
    public BookingResponse(boolean success, String reason, String bookingId) {
        this.success = success;
        this.reason = reason;
        this.bookingId = bookingId;
    }
    
    // Getters
}
```

---

### Example 2: Banking Transfer

**Problem**: Transfer money between accounts without losing or duplicating funds.

**Solution: Two-Phase Commit with Compensating Actions**

```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.*;
import java.util.concurrent.locks.ReentrantLock;

@Service
public class BankTransferService {
    private final Map<String, ReentrantLock> accountLocks = new HashMap<>();
    
    public TransferResponse transfer(String fromAccount, String toAccount, double amount) {
        String transactionId = UUID.randomUUID().toString();
        
        try {
            // Start distributed transaction
            executeDistributedTransaction(transactionId, () -> {
                // Debit from source
                debitAccount(fromAccount, amount, transactionId);
                
                // Credit to destination
                creditAccount(toAccount, amount, transactionId);
            });
            
            // Both succeed or both fail
            return new TransferResponse(true, null, transactionId);
            
        } catch (InsufficientFundsException e) {
            return new TransferResponse(false, "insufficient_funds", null);
            
        } catch (Exception e) {
            // Automatic rollback on any failure
            logFailedTransaction(transactionId, e);
            return new TransferResponse(false, "system_error", null);
        }
    }
    
    private void debitAccount(String accountId, double amount, String txId) 
            throws InsufficientFundsException {
        // Use pessimistic lock
        ReentrantLock lock = accountLocks.computeIfAbsent(
            accountId, 
            k -> new ReentrantLock()
        );
        
        lock.lock();
        try {
            double balance = getBalance(accountId);
            if (balance < amount) {
                throw new InsufficientFundsException();
            }
            
            updateBalance(accountId, balance - amount, txId);
        } finally {
            lock.unlock();
        }
    }
    
    private void creditAccount(String accountId, double amount, String txId) {
        // Credit implementation
    }
    
    @Transactional
    private void executeDistributedTransaction(String txId, Runnable operation) {
        operation.run();
    }
}

class TransferResponse {
    private final boolean success;
    private final String reason;
    private final String transactionId;
    
    public TransferResponse(boolean success, String reason, String transactionId) {
        this.success = success;
        this.reason = reason;
        this.transactionId = transactionId;
    }
    
    // Getters
}
```

---

### Example 3: E-commerce Inventory

**Problem**: Multiple customers adding same item to cart and checking out.

**Solution: Reserved Inventory with Timeout**

```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.LocalDateTime;
import java.util.*;

@Service
public class InventoryService {
    private final ReservationRepository reservationRepository;
    private final ProductRepository productRepository;
    private final OrderRepository orderRepository;
    
    public CartResponse addToCart(String productId, int quantity, String userId) {
        // Check available inventory (not reserved)
        int available = getAvailableQuantity(productId);
        
        if (available < quantity) {
            // Suggest alternative
            List<Product> alternatives = suggestSimilarProducts(productId);
            return new CartResponse(false, "insufficient_stock", null, alternatives);
        }
        
        // Soft reserve (15 minute timeout)
        Reservation reservation = createReservation(
            productId,
            quantity,
            userId,
            LocalDateTime.now().plusMinutes(15)
        );
        
        return new CartResponse(true, null, reservation.getId(), null);
    }
    
    @Transactional
    public CheckoutResponse checkout(String cartId, String userId) {
        List<Reservation> reservations = getCartReservations(cartId);
        
        // Validate all reservations still valid
        for (Reservation res : reservations) {
            if (res.isExpired()) {
                return new CheckoutResponse(false, "reservation_expired", null);
            }
        }
        
        // Convert reservations to firm commitments
        for (Reservation res : reservations) {
            commitReservation(res.getId());
            decrementInventory(res.getProductId(), res.getQuantity());
        }
        
        Order order = createOrder(userId, reservations);
        
        return new CheckoutResponse(true, null, order.getId());
    }
    
    public void releaseExpiredReservations() {
        // Background job runs every minute
        List<Reservation> expired = findExpiredReservations();
        for (Reservation reservation : expired) {
            deleteReservation(reservation.getId());
        }
    }
}

class CartResponse {
    private final boolean success;
    private final String reason;
    private final String reservationId;
    private final List<Product> alternatives;
    
    public CartResponse(boolean success, String reason, 
                       String reservationId, List<Product> alternatives) {
        this.success = success;
        this.reason = reason;
        this.reservationId = reservationId;
        this.alternatives = alternatives;
    }
    
    // Getters
}

class CheckoutResponse {
    private final boolean success;
    private final String reason;
    private final String orderId;
    
    public CheckoutResponse(boolean success, String reason, String orderId) {
        this.success = success;
        this.reason = reason;
        this.orderId = orderId;
    }
    
    // Getters
}
```

---

## Trade-offs and When to Use

### Decision Matrix

| Scenario | Recommended Approach | Why |
|----------|---------------------|-----|
| **Low Contention** | Optimistic Locking | Minimal overhead, good performance |
| **High Contention** | Pessimistic Locking or Queue | Reduces retries, more predictable |
| **Read-Heavy** | Optimistic Locking | Allows concurrent reads |
| **Write-Heavy** | Queue + Background Processing | Serializes writes efficiently |
| **Strong Consistency Required** | 2PC or Distributed Lock | Guarantees atomicity |
| **Availability > Consistency** | Saga Pattern | Eventual consistency, no blocking |
| **Single Database** | Database Transactions + Row Locks | Built-in, reliable |
| **Microservices** | Saga Pattern | Each service maintains autonomy |
| **Financial Transactions** | 2PC or Compensating Transactions | Audit trail, no lost money |
| **Ticket/Seat Booking** | Optimistic + Reservation Pattern | Handles spikes, fair allocation |

---

### Performance Characteristics

```
Throughput vs Contention Level

High  │     Optimistic
      │       Locking
      │         ╲
      │          ╲
T     │           ╲
h     │            ╲    Queue
r     │             ╲  ╱
o     │              ╲╱
u     │              ╱╲
g     │             ╱  ╲
h     │            ╱    ╲
p     │           ╱      ╲
u     │     Pessimistic   ╲
t     │       Locking      ╲
      │                     ╲
Low   │                      ╲
      └───────────────────────────────→
      Low          High          Very High
            Contention Level
```

---

### Guidelines

**Use Optimistic Locking when**:
- Conflicts are rare (< 5% collision rate)
- Reads greatly outnumber writes
- Retry logic is acceptable
- Example: User profile updates, blog comments

**Use Pessimistic Locking when**:
- Conflicts are frequent (> 20% collision rate)
- Cost of retry is high
- Need guaranteed single access
- Example: Seat selection, critical counter updates

**Use Distributed Locks when**:
- Multiple services access shared resource
- Need mutual exclusion across nodes
- Can tolerate lock service dependency
- Example: Scheduled job coordination, leader election

**Use Saga Pattern when**:
- Long-running business processes
- Spanning multiple microservices
- Availability more important than immediate consistency
- Compensation is possible
- Example: Order fulfillment, travel booking

**Use 2PC when**:
- Strong consistency is non-negotiable
- Atomicity across systems required
- Can accept performance cost
- Example: Financial transfers, regulatory compliance

---

## Best Practices

### 1. Always Set Timeouts
```java
import java.util.concurrent.TimeUnit;

// Don't wait forever for locks
try {
    if (!lock.tryLock(5, TimeUnit.SECONDS)) {
        return errorResponse("resource_busy");
    }
    try {
        // Critical section
    } finally {
        lock.unlock();
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    return errorResponse("interrupted");
}
```

### 2. Implement Idempotency
```java
import java.util.concurrent.ConcurrentHashMap;

public class PaymentService {
    private final ConcurrentHashMap<String, PaymentResult> payments = 
        new ConcurrentHashMap<>();
    
    public PaymentResult processPayment(String paymentId, double amount, 
                                       String idempotencyKey) {
        // Check if already processed
        PaymentResult existing = payments.get(idempotencyKey);
        if (existing != null) {
            return existing;
        }
        
        // Process new payment
        PaymentResult result = chargeCard(amount);
        payments.put(idempotencyKey, result);
        return result;
    }
}
```

### 3. Use Exponential Backoff
```java
import java.util.Random;
import java.util.concurrent.Callable;
import java.util.concurrent.TimeUnit;

public class RetryHelper {
    private static final int MAX_RETRIES = 5;
    private static final Random random = new Random();
    
    public <T> T retryWithBackoff(Callable<T> operation) throws Exception {
        for (int attempt = 0; attempt < MAX_RETRIES; attempt++) {
            try {
                return operation.call();
            } catch (ConflictException e) {
                if (attempt == MAX_RETRIES - 1) {
                    throw e;
                }
                long sleepTime = (long) (Math.pow(2, attempt) * 100); // milliseconds
                long jitter = random.nextInt(100);
                TimeUnit.MILLISECONDS.sleep(sleepTime + jitter);
            }
        }
        throw new RuntimeException("Max retries exceeded");
    }
}
```

### 4. Monitor Contention Metrics
```java
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;

public class ContentionMonitor {
    private final MeterRegistry meterRegistry;
    
    public ContentionMonitor(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }
    
    public void trackRetry(String resource) {
        // Track retry rates
        meterRegistry.counter("contention.retry", 
            "resource", resource).increment();
    }
    
    public void trackLockWaitTime(Runnable lockOperation) {
        // Track lock wait times
        Timer.Sample sample = Timer.start(meterRegistry);
        lockOperation.run();
        sample.stop(meterRegistry.timer("lock.wait_time"));
    }
    
    public void checkContentionThreshold(double retryRate, double threshold) {
        // Alert on high contention
        if (retryRate > threshold) {
            alert("High contention on resource");
        }
    }
}
```

### 5. Design for Failure
```java
// Always have compensation logic
public class OrderProcessor {
    
    public void processOrder(Order order) throws OrderException {
        boolean inventoryReserved = false;
        boolean paymentCharged = false;
        
        try {
            reserveInventory(order);
            inventoryReserved = true;
            
            chargePayment(order);
            paymentCharged = true;
            
            sendConfirmation(order);
            
        } catch (PaymentException e) {
            // Compensate: release inventory
            if (inventoryReserved) {
                releaseInventory(order);
            }
            throw e;
            
        } catch (Exception e) {
            // Compensate: rollback everything
            if (paymentCharged) {
                refundPayment(order);
            }
            if (inventoryReserved) {
                releaseInventory(order);
            }
            throw new OrderException("Order processing failed", e);
        }
    }
}
```

---

## Summary

**Contention** is inevitable in distributed systems with shared resources. The key is choosing the right strategy based on:

1. **Consistency requirements**: How strict?
2. **Performance needs**: Throughput vs latency
3. **Failure tolerance**: Can you retry?
4. **System architecture**: Monolith vs microservices

**Remember**: 
- ⚡ **Optimistic** for low contention
- 🔒 **Pessimistic** for high contention
- 🔄 **Saga** for distributed workflows
- 🎯 **2PC** for strong consistency (use sparingly)

The best solution often combines multiple approaches: optimistic locking for normal cases with fallback to queuing under high load, plus compensation logic for failures.
