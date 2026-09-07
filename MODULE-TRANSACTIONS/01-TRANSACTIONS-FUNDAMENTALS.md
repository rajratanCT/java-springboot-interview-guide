# Transactions in Spring Boot - Complete Guide

## What is a Transaction?

A transaction is a sequence of database operations that are executed as a single, atomic unit. Either all operations complete successfully, or none of them do.

### Real-World Analogy:
```
Banking Transfer (Person A sends $100 to Person B):

1. Debit $100 from Person A's account
2. Credit $100 to Person B's account

If step 1 succeeds but step 2 fails:
- Person A loses $100 (money disappears)
- Person B doesn't receive anything
- System is in inconsistent state (DATA LOSS!)

Transaction ensures:
- Both steps complete successfully, OR
- Both steps roll back (no changes)
- System always remains consistent
```

---

## ACID Properties

Transactions guarantee ACID properties:

### 1. **Atomicity** (All or Nothing)
```
DEFINITION:
All operations in a transaction must complete successfully.
If any operation fails, entire transaction rolls back.
No partial updates.

EXAMPLE:
@Transactional
public void transferMoney(Long fromAccountId, Long toAccountId, BigDecimal amount) {
    // Operation 1
    Account fromAccount = accountRepository.findById(fromAccountId).orElseThrow();
    fromAccount.debit(amount);  // Debit from source
    accountRepository.save(fromAccount);
    
    // Simulate error
    if (toAccountId == -1) {
        throw new AccountNotFoundException("Recipient not found");
    }
    
    // Operation 2
    Account toAccount = accountRepository.findById(toAccountId).orElseThrow();
    toAccount.credit(amount);   // Credit to destination
    accountRepository.save(toAccount);
}

SCENARIO:
- Debit succeeds ✓
- Exception thrown before credit
- ENTIRE TRANSACTION ROLLS BACK ✓
- Source account returns to original balance
- No partial transfer!
```

### 2. **Consistency** (Valid State)
```
DEFINITION:
Database moves from one valid state to another.
All business rules and constraints are maintained.
No invalid data states allowed.

EXAMPLE:
Business Rules:
- Account balance can never be negative
- Total money in system = constant
- Every credit must have corresponding debit

@Transactional
public void transferMoney(Long fromId, Long toId, BigDecimal amount) {
    // BEFORE TRANSACTION
    // Account A: $1000
    // Account B: $500
    // Total: $1500
    
    // DURING TRANSACTION
    debitAccount(fromId, amount);      // Account A: $900
    creditAccount(toId, amount);       // Account B: $600
    
    // AFTER TRANSACTION
    // Account A: $900
    // Account B: $600
    // Total: $1500 (CONSISTENT!)
    // All rules maintained ✓
}
```

### 3. **Isolation** (Concurrent Independence)
```
DEFINITION:
Concurrent transactions don't interfere with each other.
Each transaction executes independently.
No dirty reads, lost updates, or phantom reads.

EXAMPLE:
Without Isolation:

Transaction 1                      Transaction 2
─────────────────────────────────────────────────
Read Account Balance: $1000        
                                   Read Account Balance: $1000
Update Balance: $900
Commit ✓
                                   Update Balance: $800
                                   Commit ✓
                                   
RESULT: Final balance = $800 (Transaction 1's update lost!)


With Isolation (Serializable):

Transaction 1                      Transaction 2
─────────────────────────────────────────────────
Read Account Balance: $1000        
Lock Account                       (Waits for lock)
Update Balance: $900
Commit ✓
Release Lock
                                   Read Account Balance: $900
                                   Lock Account
                                   Update Balance: $800
                                   Commit ✓
                                   
RESULT: Final balance = $800 (Correct, both updates applied sequentially)
```

### 4. **Durability** (Permanent)
```
DEFINITION:
Once transaction commits, changes are permanent.
Survives system crashes, power failures, etc.
Data written to persistent storage.

EXAMPLE:
@Transactional
public void transferMoney(Long fromId, Long toId, BigDecimal amount) {
    // Operations...
    accountRepository.save(accountA);
    accountRepository.save(accountB);
    
    // Transaction COMMITS here
    // ✓ Data written to disk
    // ✓ Even if server crashes now, changes persist
    // ✓ Changes survive power failure
}
```

---

## Transaction Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│                    TRANSACTION LIFECYCLE                      │
└─────────────────────────────────────────────────────────────┘

1. BEGIN TRANSACTION
   └─ Open database connection
   └─ Start transaction
   └─ Set isolation level

2. EXECUTE OPERATIONS
   ├─ Read/Write operations
   ├─ Create savepoints (optional)
   └─ Accumulate changes in memory

3. COMMIT or ROLLBACK
   ├─ COMMIT: Write all changes permanently to database
   └─ ROLLBACK: Discard all changes, return to initial state

4. END TRANSACTION
   └─ Close database connection
   └─ Release locks


CODE EXAMPLE:

@Transactional
public void processTransaction() {
    // ↓ BEGIN TRANSACTION
    System.out.println("Transaction started");
    
    // ↓ EXECUTE OPERATIONS
    User user = userRepository.findById(1L).orElseThrow();
    user.setBalance(user.getBalance() - 100);
    userRepository.save(user);
    
    // ↓ If exception here, ROLLBACK occurs
    if (user.getBalance() < 0) {
        throw new InsufficientFundsException();
    }
    
    // ↓ COMMIT occurs here (method ends successfully)
    System.out.println("Transaction committed");
    // ↑ END TRANSACTION
}
```

---

## Isolation Levels

Define how concurrent transactions interact:

### 1. **READ_UNCOMMITTED** (Lowest)
```
DEFINITION:
Transaction can read uncommitted (dirty) data from other transactions.

PROBLEMS:
✗ Dirty reads: Read data another transaction might rollback
✗ Lost updates: Updates overwritten
✗ Phantom reads: Data appears/disappears

EXAMPLE:
Transaction A                          Transaction B
─────────────────────────────────────────────────────
BEGIN
Read Account Balance: $1000
                                      BEGIN
                                      Update Balance: $500
                                      (Not yet committed)
Read Account Balance: $500 ✗ DIRTY READ!
                                      ROLLBACK ✗
Balance back to $1000
But Transaction A saw $500!

WHEN TO USE:
- Never for critical operations
- Performance over consistency not worth it
```

### 2. **READ_COMMITTED** (Default for most)
```
DEFINITION:
Transaction only reads committed data.
Can see updates from other committed transactions.

PROBLEMS:
✗ Non-repeatable reads: Same query returns different results
✗ Phantom reads: Rows appear/disappear between queries

EXAMPLE:
Transaction A                          Transaction B
─────────────────────────────────────────────────────
BEGIN
SELECT balance FROM account: $1000
                                      BEGIN
                                      UPDATE balance = $500
                                      COMMIT ✓
SELECT balance FROM account: $500
(Non-repeatable read - value changed!)

WHEN TO USE:
- Default for most applications
- Good balance of performance and safety
- Web applications, e-commerce
```

### 3. **REPEATABLE_READ**
```
DEFINITION:
Transaction gets consistent snapshot of data.
Cannot see new rows inserted by others (phantom reads may occur).

PROBLEMS:
✗ Phantom reads: New rows appear between queries

EXAMPLE:
Transaction A                          Transaction B
─────────────────────────────────────────────────────
BEGIN
SELECT COUNT(*) FROM orders: 5
                                      BEGIN
                                      INSERT INTO orders VALUES (...)
                                      COMMIT ✓
SELECT COUNT(*) FROM orders: 6
(Phantom read - new row appeared!)

WHEN TO USE:
- Financial applications
- Reports generation
- MySQL InnoDB default
```

### 4. **SERIALIZABLE** (Highest)
```
DEFINITION:
Transactions execute completely independently.
As if one executes completely before the other.
Prevents all concurrency issues.

PROBLEMS:
✗ Severe performance impact (slow)
✗ Potential deadlocks
✗ Low throughput

EXAMPLE:
Transaction A                          Transaction B
─────────────────────────────────────────────────────
BEGIN
LOCK TABLE orders
SELECT * FROM orders
                                      BEGIN
                                      (WAITS for lock) ⏳
UPDATE orders SET ...
COMMIT
RELEASE LOCK
                                      (Lock acquired) ✓
                                      (Can proceed)

WHEN TO USE:
- Critical financial operations
- Regulatory requirements
- When absolute isolation required
```

### Comparison Table:

| Level | Dirty Read | Non-Repeatable | Phantom | Performance |
|-------|-----------|-----------------|---------|-------------|
| READ_UNCOMMITTED | ✓ | ✓ | ✓ | ⚡⚡⚡ |
| READ_COMMITTED | ✗ | ✓ | ✓ | ⚡⚡ |
| REPEATABLE_READ | ✗ | ✗ | ✓ | ⚡ |
| SERIALIZABLE | ✗ | ✗ | ✗ | 🐌 |

---

## Propagation Types

Define how transactions behave when methods call other transactional methods:

### 1. **REQUIRED** (Default)
```
DEFINITION:
Use existing transaction if available, else create new one.

EXAMPLE:

@Transactional(propagation = Propagation.REQUIRED)
public void methodA() {
    // BEGIN TRANSACTION_A
    methodB(); // Uses TRANSACTION_A
    // COMMIT TRANSACTION_A
}

@Transactional(propagation = Propagation.REQUIRED)
public void methodB() {
    // Uses existing TRANSACTION_A (doesn't create new)
}

SCENARIO 1: methodA() calls methodB()
TX1 BEGIN
  TX1: methodA() executes
  TX1: methodB() executes (same transaction)
TX1 COMMIT

SCENARIO 2: methodB() called directly
TX2 BEGIN
  TX2: methodB() executes (new transaction created)
TX2 COMMIT
```

### 2. **REQUIRES_NEW**
```
DEFINITION:
Always creates new transaction.
Suspends current transaction if exists.

EXAMPLE:

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void logActivity(String message) {
    // Always new transaction
    // Even if called from another transactional method
}

@Transactional
public void processOrder(Order order) {
    // TX1 BEGIN
    orderRepository.save(order);
    
    // TX1 SUSPENDED
    // TX2 BEGIN (new transaction)
    logActivity("Order processed: " + order.getId());
    // TX2 COMMIT
    // TX1 RESUMED
    
    // TX1 COMMIT
}

BENEFIT: Log persists even if order processing fails!
```

### 3. **SUPPORTS**
```
DEFINITION:
Use transaction if available, but doesn't require one.

EXAMPLE:

@Transactional(propagation = Propagation.SUPPORTS)
public User getUser(Long id) {
    // Read-only, no transaction needed
    return userRepository.findById(id).orElse(null);
}

SCENARIO 1: Called from transactional method
Uses existing transaction ✓

SCENARIO 2: Called from non-transactional code
Executes without transaction ✓ (no overhead)
```

### 4. **NOT_SUPPORTED**
```
DEFINITION:
Never uses transaction.
Suspends current transaction if exists.

EXAMPLE:

@Transactional(propagation = Propagation.NOT_SUPPORTED)
public void sendNotification(String message) {
    // No transaction
    // Send email/SMS (external system)
    // If it fails, doesn't affect main transaction
}

BENEFIT: External calls don't hold database locks
```

### 5. **MANDATORY**
```
DEFINITION:
Must be called from existing transaction.
Throws exception if no transaction exists.

EXAMPLE:

@Transactional(propagation = Propagation.MANDATORY)
public void internalMethod() {
    // Must have active transaction
}

@Transactional
public void externalMethod() {
    internalMethod(); // ✓ Works (has transaction)
}

public void main() {
    internalMethod(); // ✗ Throws TransactionRequiredException
}
```

### 6. **NEVER**
```
DEFINITION:
Must NOT be called from transaction.
Throws exception if transaction exists.

EXAMPLE:

@Transactional(propagation = Propagation.NEVER)
public void readConfig() {
    // Read config from file (no transaction needed)
}

@Transactional
public void processOrder() {
    readConfig(); // ✗ Throws TransactionRequiredException
}
```

### 7. **NESTED**
```
DEFINITION:
Uses savepoint within same transaction.
If nested fails, can rollback just that part.

EXAMPLE:

@Transactional
public void mainProcess() {
    // TX BEGIN
    updateInventory();
    
    // SAVEPOINT_1 created
    try {
        processPayment();
    } catch (PaymentException e) {
        // ROLLBACK to SAVEPOINT_1 only
        // Inventory update remains!
    }
    // TX COMMIT
}

@Transactional(propagation = Propagation.NESTED)
public void processPayment() {
    // Uses savepoint
}

BENEFIT: Partial rollback within same transaction
```

---

## Practical Transaction Scenarios

### Scenario 1: Money Transfer
```java
@Service
public class TransferService {
    
    @Autowired
    private AccountRepository accountRepository;
    
    @Transactional
    public void transferMoney(Long fromId, Long toId, BigDecimal amount) {
        // Step 1: Debit source
        Account source = accountRepository.findById(fromId)
            .orElseThrow(() -> new AccountNotFoundException());
        
        if (source.getBalance().compareTo(amount) < 0) {
            throw new InsufficientFundsException();
        }
        
        source.setBalance(source.getBalance().subtract(amount));
        accountRepository.save(source);
        
        // Step 2: Credit destination
        Account destination = accountRepository.findById(toId)
            .orElseThrow(() -> new AccountNotFoundException());
        
        destination.setBalance(destination.getBalance().add(amount));
        accountRepository.save(destination);
        
        // If any exception, both operations roll back
        // No partial transfer!
    }
}

TEST:
// Before: Account A = $1000, Account B = $500
transferService.transferMoney(1L, 2L, new BigDecimal("100"));
// After: Account A = $900, Account B = $600 ✓

// Test failure scenario:
// Before: Account A = $1000, Account B = $500
transferService.transferMoney(1L, 999L, new BigDecimal("100")); // Account 999 doesn't exist
// Exception thrown after debit but before credit
// ROLLBACK occurs
// After: Account A = $1000, Account B = $500 ✓ (unchanged)
```

### Scenario 2: Order Processing with Multiple Operations
```java
@Service
public class OrderService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private PaymentService paymentService;
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        // Operation 1: Create order
        Order order = new Order();
        order.setCustomerId(request.getCustomerId());
        order.setStatus("PENDING");
        order = orderRepository.save(order);
        
        // Operation 2: Reserve inventory
        for (OrderItem item : request.getItems()) {
            inventoryService.reserveStock(item.getProductId(), item.getQuantity());
        }
        
        // Operation 3: Process payment
        paymentService.processPayment(request.getCustomerId(), order.getTotal());
        
        // Operation 4: Confirm order
        order.setStatus("CONFIRMED");
        order = orderRepository.save(order);
        
        return order;
    }
}

TRANSACTION FLOW:
┌─────────────────────────────────────────┐
│ BEGIN TRANSACTION                       │
├─────────────────────────────────────────┤
│ Create Order (PENDING)                  │
│ Reserve Inventory                       │
│ Process Payment                         │
│ Update Order (CONFIRMED)                │
├─────────────────────────────────────────┤
│ COMMIT (all changes persist)            │
└─────────────────────────────────────────┘

IF ERROR AT ANY STEP:
┌─────────────────────────────────────────┐
│ BEGIN TRANSACTION                       │
├─────────────────────────────────────────┤
│ Create Order (PENDING)           ✓      │
│ Reserve Inventory                ✓      │
│ Process Payment        ✗ EXCEPTION      │
├─────────────────────────────────────────┤
│ ROLLBACK ALL                            │
│ Order not created                       │
│ Inventory not reserved                  │
└─────────────────────────────────────────┘
```

### Scenario 3: Savepoints (Nested Transactions)
```java
@Service
public class ComplexService {
    
    @Transactional
    public void complexProcess() {
        step1(); // Always succeeds
        
        try {
            step2(); // May fail
        } catch (Exception e) {
            // Rollback only step2, keep step1
            log.error("Step2 failed, continuing with step3", e);
        }
        
        step3(); // Always executes
    }
    
    @Transactional(propagation = Propagation.NESTED)
    public void step2() {
        // If this fails, only this part rolls back
    }
}
```

---

## Common Mistakes

### ❌ Mistake 1: Missing @Transactional
```java
// WRONG: No transaction
public void transferMoney(Long fromId, Long toId, BigDecimal amount) {
    Account source = repo.findById(fromId);
    source.setBalance(source.getBalance().subtract(amount));
    repo.save(source);
    
    Account dest = repo.findById(toId);
    dest.setBalance(dest.getBalance().add(amount));
    repo.save(dest);
    // If exception after first save, no rollback!
}

// CORRECT: With transaction
@Transactional
public void transferMoney(Long fromId, Long toId, BigDecimal amount) {
    // Both saves are atomic
}
```

### ❌ Mistake 2: Catching Exception in @Transactional
```java
// WRONG: Exception caught, transaction won't rollback
@Transactional
public void process() {
    try {
        risky operation();
    } catch (Exception e) {
        // Exception handled, @Transactional thinks success
        log.error("Error", e);
    }
}

// CORRECT: Let exception propagate
@Transactional
public void process() {
    risky operation(); // Exception propagates, triggers rollback
}

// OR: Throw custom exception
@Transactional
public void process() {
    try {
        risky operation();
    } catch (Exception e) {
        throw new BusinessException("Processing failed", e);
    }
}
```

### ❌ Mistake 3: Self-invocation
```java
// WRONG: Transaction not applied
@Service
public class UserService {
    
    public void register(User user) {
        saveUser(user); // Transaction NOT applied! (self-call)
    }
    
    @Transactional
    public void saveUser(User user) {
        // @Transactional ignored when called via self
        userRepository.save(user);
    }
}

// CORRECT: Inject separate service
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Transactional
    public void register(User user) {
        userRepository.save(user); // Transaction applied
    }
}
```

### ❌ Mistake 4: Long-running transactions
```java
// WRONG: Very long transaction
@Transactional
public void processAllOrders() {
    List<Order> orders = orderRepository.findAll(); // 100,000 orders
    
    for (Order order : orders) {
        // Process and update each order
        // PROBLEM: Single transaction for hours!
        // Locks held entire time
        // Memory bloat
        // Performance impact
    }
}

// CORRECT: Batch processing
@Transactional
public void processAllOrders() {
    int pageSize = 100;
    int page = 0;
    
    while (true) {
        List<Order> orders = orderRepository.findAll(
            PageRequest.of(page, pageSize)
        );
        
        if (orders.isEmpty()) break;
        
        // Each page in separate transaction
        processOrderBatch(orders);
        page++;
    }
}

@Transactional
private void processOrderBatch(List<Order> orders) {
    // Process small batch
    // Transaction commits quickly
    // Locks released immediately
}
```

### ❌ Mistake 5: Unchecked exceptions not triggering rollback
```java
// Spring by default only rollbacks on RuntimeException
// Be explicit for checked exceptions

@Transactional(rollbackFor = Exception.class)
public void process() throws Exception {
    // Now rolls back on ANY exception (checked or unchecked)
}

@Transactional(noRollbackFor = ValidationException.class)
public void process() {
    // Doesn't rollback on ValidationException
    // Useful for expected errors that shouldn't trigger rollback
}
```

---

## Summary Table

| Concept | Purpose | Key Point |
|---------|---------|----------|
| **Atomicity** | All or nothing | Complete success or complete rollback |
| **Consistency** | Valid state | Database constraints maintained |
| **Isolation** | Independence | Concurrent transactions isolated |
| **Durability** | Permanent | Committed changes survive failures |
| **READ_UNCOMMITTED** | Fastest | Not recommended for production |
| **READ_COMMITTED** | Default | Good for most apps |
| **REPEATABLE_READ** | Consistent reads | MySQL default |
| **SERIALIZABLE** | Safest | Slowest, rarely needed |
| **REQUIRED** | Standard | Use existing or create new |
| **REQUIRES_NEW** | Independent | New transaction always |
| **NESTED** | Savepoint | Partial rollback possible |

---

This covers all the fundamental concepts of transactions in Spring Boot!