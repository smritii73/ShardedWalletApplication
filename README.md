# Wallet App - Money Transfer System

A Spring Boot application for handling money transfers between users with saga pattern for distributed transactions. Includes database sharding for scalability.

## Overview

This wallet application is a realistic payment system backend that handles:
- User account creation and management
- Wallet creation with balance tracking
- Money transfers between users with multi-step transactions
- Automatic rollback if transfer fails mid-process
- Data distribution across multiple databases using sharding
- Concurrent transaction safety with pessimistic locking

Think of it like PayPal or Google Pay's backend - when you transfer money, multiple things happen in sequence, and if any step breaks, the system automatically reverses previous steps to keep your money safe.

## What This Does

Users can transfer money from one wallet to another. The system uses a saga pattern to handle multi-step transactions reliably, even if something fails mid-transfer.

**Example**: User A transfers $100 to User B
1. Deduct $100 from User A's wallet (lock wallet, check balance, subtract)
2. Add $100 to User B's wallet (lock wallet, add amount)
3. Mark transaction as SUCCESS (update transaction status)
4. If any step fails → automatic compensation (rollback all previous steps)

The entire process is tracked in database with saga instances and saga steps, so you can see exactly what happened at each stage.

## Quick Start

### Prerequisites
- Java 17+
- MySQL 8+
- Two MySQL databases (for sharding): `realshard1` and `realshard2`

### Setup

1. **Create Databases**
```sql
CREATE DATABASE realshard1;
CREATE DATABASE realshard2;
```

2. **Configure Database Connection**
Edit `src/main/resources/sharding.yml`:
```yaml
dataSources:
  realshard1:
    jdbcUrl: jdbc:mysql://localhost:3306/realshard1?allowPublicKeyRetrieval=true&useSSL=false
    username: root
    password: root@123

  realshard2:
    jdbcUrl: jdbc:mysql://localhost:3306/realshard2?allowPublicKeyRetrieval=true&useSSL=false
    username: root
    password: root@123
```

3. **Run Application**
```bash
mvn clean install
mvn spring-boot:run
```

App runs on `http://localhost:8080`

4. **Verify Setup**
```bash
# Check if app started successfully
curl http://localhost:8080/api/v1/user/id/1
# Should return 404 (no user) or 500 (if DB issue) - either means app is running
```

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Client / API Request                         │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
        ┌─────────────────────────────────────┐
        │   TransactionController             │
        │   (Handle /api/v1/transaction)      │
        │                                     │
        │   1. Validate request               │
        │   2. Create transaction record      │
        │   3. Initiate saga                  │
        └──────────────┬──────────────────────┘
                       │
                       ▼
        ┌─────────────────────────────────────┐
        │  TransferSagaService                │
        │  (Orchestrate multi-step transfer)  │
        │                                     │
        │  1. Create SagaContext (shared data)│
        │  2. Create SagaInstance (track it)  │
        │  3. Execute steps in sequence       │
        │  4. Handle failures with rollback   │
        └──────────────┬──────────────────────┘
                       │
        ┌──────────────┴────────────────────────────────┐
        │                                               │
        ▼                                               ▼
  Step 1: DebitSourceWalletStep              Step 2: CreditDestinationWalletStep
  ├─ Get source wallet with LOCK            ├─ Get destination wallet with LOCK
  ├─ Check sufficient balance               ├─ Add amount to balance
  ├─ Debit the amount                       ├─ Save wallet
  ├─ Save wallet                            └─ Store snapshot in SagaContext
  └─ Store snapshot in SagaContext
        │                                               │
        └──────────────┬────────────────────────────────┘
                       │
                       ▼
        ┌─────────────────────────────────────┐
        │  Step 3: UpdateTransactionStatus    │
        │  ├─ Get transaction by ID           │
        │  ├─ Update status to SUCCESS        │
        │  └─ Save transaction                │
        └──────────────┬──────────────────────┘
                       │
          ┌────────────┴────────────┐
          │                         │
       SUCCESS                    FAILURE
          │                         │
          ▼                         ▼
    Mark saga as              Mark saga as
    COMPLETED                 FAILED
          │                         │
          │                         ├─ Mark SagaInstance status
          │                         │
          │                         └─ Start COMPENSATION:
          │                            └─ Compensate Step 2 (reverse credit)
          │                            └─ Compensate Step 1 (reverse debit)
          │
          ▼
    ┌─────────────────────────────────┐
    │   ShardingSphere Router         │
    │   (Transparent sharding)        │
    │                                 │
    │  Shard key: user_id             │
    │  Route: realshard1 or 2         │
    └─────────────────────────────────┘
          │
    ┌─────┴──────┐
    │            │
    ▼            ▼
  MySQL1       MySQL2
realshard1   realshard2
```

## Data Flow: Money Transfer in Detail

Let's trace a real transfer: **Alice ($500) → Bob ($0) : Transfer $100**

```
REQUEST:
POST /api/v1/transaction
{
  "fromUserId": 1,
  "toUserId": 2,
  "amount": 100.00,
  "description": "Coffee money"
}

STEP 1: TransactionController receives request
├─ Validate input (userId exists? amount > 0?)
└─ Call TransferSagaService.initiateTransfer()

STEP 2: TransferSagaService.initiateTransfer()
├─ Create Transaction record (status: PENDING)
│  Transaction {
│    id: 123,
│    fromUserId: 1,
│    toUserId: 2,
│    amount: 100.00,
│    status: PENDING,
│    sagaInstanceId: null
│  }
├─ Create SagaContext (shared data for all steps)
│  {
│    transactionId: 123,
│    fromUserId: 1,
│    toUserId: 2,
│    amount: 100.00,
│    description: "Coffee money",
│    destinationTransactionStatus: "SUCCESS"
│  }
├─ Create SagaInstance
│  SagaInstance {
│    id: 456,
│    status: STARTED,
│    context: "{...json...}",
│    currentStep: null
│  }
└─ Update Transaction with sagaInstanceId

STEP 3: Execute Step 1 - DebitSourceWalletStep
├─ Get Alice's wallet WITH PESSIMISTIC LOCK
│  Wallet {
│    userId: 1,
│    balance: 500.00
│  }
│  [LOCKED - only this transaction can touch it]
├─ Store original balance in context
│  sagaContext.put("originalSourceWalletBalance", 500.00)
├─ Validate sufficient balance (500 >= 100 ✓)
├─ Debit the amount
│  wallet.debit(100.00)  // balance = 400.00
├─ Save to database
│  Wallet { userId: 1, balance: 400.00 }
├─ Store new balance in context
│  sagaContext.put("sourceWalletBalanceAfterDebit", 400.00)
├─ Mark SagaStep as COMPLETED
└─ [UNLOCK Alice's wallet]

STEP 4: Execute Step 2 - CreditDestinationWalletStep
├─ Get Bob's wallet WITH PESSIMISTIC LOCK
│  Wallet {
│    userId: 2,
│    balance: 0.00
│  }
│  [LOCKED - only this transaction can touch it]
├─ Store original balance in context
│  sagaContext.put("OriginalDestinationWalletBalance", 0.00)
├─ Credit the amount
│  wallet.credit(100.00)  // balance = 100.00
├─ Save to database
│  Wallet { userId: 2, balance: 100.00 }
├─ Store new balance in context
│  sagaContext.put("DestinationWalletBalanceAfterCredit", 100.00)
├─ Mark SagaStep as COMPLETED
└─ [UNLOCK Bob's wallet]

STEP 5: Execute Step 3 - UpdateTransactionStatus
├─ Get Transaction from database
│  Transaction { id: 123, status: PENDING }
├─ Set status from SagaContext
│  transaction.status = TransactionStatus.SUCCESS
├─ Save to database
│  Transaction { id: 123, status: SUCCESS }
├─ Store status in context
│  sagaContext.put("transactionStatusAfterUpdate", SUCCESS)
└─ Mark SagaStep as COMPLETED

COMPLETION:
├─ All steps COMPLETED
├─ Mark SagaInstance as COMPLETED
└─ Update SagaInstance context with final snapshots

FINAL STATE:
├─ Alice's wallet: 400.00 (originally 500)
├─ Bob's wallet: 100.00 (originally 0)
├─ Transaction: SUCCESS
└─ SagaInstance: COMPLETED with full audit trail
```

### What If Something Fails?

```
Scenario: Database connection lost during Step 3

STEP 1: Debit source ✓ COMPLETED
        Alice: 500 → 400

STEP 2: Credit destination ✓ COMPLETED
        Bob: 0 → 100

STEP 3: Update status ✗ FAILED (connection lost)
        Transaction still PENDING

AUTOMATIC COMPENSATION (ROLLBACK):

COMPENSATE STEP 2:
├─ Get Bob's wallet with LOCK
│  Current balance: 100.00
├─ Store current balance
│  sagaContext.put("DestinationWalletBalanceBeforeCreditCompensation", 100.00)
├─ Reverse the credit (subtract amount)
│  wallet.debit(100.00)  // balance = 0.00
├─ Save to database
│  Wallet { userId: 2, balance: 0.00 }
├─ Store compensated balance
│  sagaContext.put("DestinationWalletBalanceAfterCreditCompensation", 0.00)
└─ Mark SagaStep as COMPENSATED

COMPENSATE STEP 1:
├─ Get Alice's wallet with LOCK
│  Current balance: 400.00
├─ Store current balance
│  sagaContext.put("sourceWalletBalanceBeforeCreditCompensation", 400.00)
├─ Reverse the debit (add amount back)
│  wallet.credit(100.00)  // balance = 500.00
├─ Save to database
│  Wallet { userId: 1, balance: 500.00 }
├─ Store compensated balance
│  sagaContext.put("sourceWalletBalanceAfterCreditCompensation", 500.00)
└─ Mark SagaStep as COMPENSATED

FINAL STATE:
├─ Alice's wallet: 500.00 (RESTORED)
├─ Bob's wallet: 0.00 (RESTORED)
├─ SagaInstance: COMPENSATED (failure handled)
└─ Transaction: PENDING (never marked as SUCCESS)
   (Client gets error, knows transfer didn't complete)
```

## API Endpoints

### Users

```bash
# Create user
POST /api/v1/user
{
  "name": "John Doe",
  "email": "john@example.com"
}

# Get user by ID
GET /api/v1/user/id/1

# Get user by name
GET /api/v1/user?name=John
```

### Wallets

```bash
# Create wallet for user
POST /api/v1/wallet
{
  "userId": 1
}

# Get wallet for user
GET /api/v1/wallet/user/1

# Get wallet balance
GET /api/v1/wallet/user/1/balance

# Add money to wallet
POST /api/v1/wallet/user/1/credit
{
  "amount": 500.00
}

# Withdraw from wallet
POST /api/v1/wallet/user/1/debit
{
  "amount": 100.00
}
```

### Transactions (Main Feature)

```bash
# Transfer money from one user to another
POST /api/v1/transaction
{
  "fromUserId": 1,
  "toUserId": 2,
  "amount": 50.00,
  "description": "Payment for coffee"
}

# Response: { "sagaInstanceId": 123 }
# Saga executes automatically - all 3 steps happen in sequence
```

## Understanding the Saga Pattern

### What is a Saga?

A saga is a design pattern for handling transactions in distributed systems. Instead of one big transaction (which doesn't work across multiple databases), a saga breaks it into smaller, independent steps that can be rolled back individually.

**Real world analogy**: Booking a vacation
- Step 1: Book flight (charge money)
- Step 2: Book hotel (charge money)
- Step 3: Book rental car (charge money)

If hotel booking fails, you want to automatically cancel flight and refund money. That's a saga with compensation!

### Why Not Just Use Database Transactions?

Database transactions (ACID) work great for a single database, but here's why we need sagas:

```
Without Saga (Wrong Approach):
┌─────────────────────────────────┐
│   START TRANSACTION             │
├─────────────────────────────────┤
│ 1. Debit Alice's wallet         │
│ 2. Credit Bob's wallet          │
│ 3. Update transaction status    │
│ 4. COMMIT                       │
└─────────────────────────────────┘
     ↓
Problem: What if Step 2 is in a DIFFERENT database?
Database transactions can't span multiple DBs!

With Saga (Right Approach):
┌─────────────────────────────────┐
│   SAGA INSTANCE CREATED         │
├─────────────────────────────────┤
│ 1. Execute: Debit Alice         │ (Track in SagaStep)
│    ✓ COMPLETED                  │
├─────────────────────────────────┤
│ 2. Execute: Credit Bob          │ (Track in SagaStep)
│    ✓ COMPLETED                  │
├─────────────────────────────────┤
│ 3. Execute: Update status       │ (Track in SagaStep)
│    ✗ FAILED                    │
├─────────────────────────────────┤
│ 4. COMPENSATION (Automatic):    │
│    - Compensate Step 2 (undo)   │
│    - Compensate Step 1 (undo)   │
│    ✓ COMPENSATED                │
└─────────────────────────────────┘

Result: No money lost, everything rolled back!
```

### Saga State Machine

```
                    ┌──────────────────┐
                    │     STARTED      │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │     RUNNING      │ (executing steps)
                    └───┬──────────┬───┘
                        │          │
                   SUCCESS       FAILURE
                        │          │
                        ▼          ▼
            ┌──────────────────┐  ┌──────────────────┐
            │   COMPLETED      │  │     FAILED       │
            └──────────────────┘  └────────┬─────────┘
                                           │
                                           ▼
                                  ┌──────────────────┐
                                  │ COMPENSATING     │ (undoing steps)
                                  └────────┬─────────┘
                                           │
                                           ▼
                                  ┌──────────────────┐
                                  │ COMPENSATED      │ (fully rolled back)
                                  └──────────────────┘
```

### Step State Machine

Each step inside a saga follows its own lifecycle:

```
┌────────────┐
│  PENDING   │ (created, waiting to execute)
└─────┬──────┘
      │
      ▼
┌────────────┐
│  RUNNING   │ (currently executing)
└─────┬──────┘
      │
   ┌──┴──┐
   │     │
SUCCESS FAILURE
   │     │
   ▼     ▼
┌────────────┐    ┌────────────┐
│COMPLETED   │    │  FAILED    │
└────────────┘    └─────┬──────┘
   (continue)           │
   to next step         ▼
                  ┌──────────────┐
                  │COMPENSATING  │
                  └────────┬─────┘
                           │
                           ▼
                  ┌──────────────┐
                  │COMPENSATED   │
                  └──────────────┘
                  (step undone)
```

### How Compensation Works

Each step must know how to undo itself:

```
DebitSourceWalletStep:

execute():
├─ Lock wallet
├─ Subtract amount
├─ Save wallet
└─ Return true (success)

compensate():
├─ Lock wallet (same wallet)
├─ Add amount back (reverse of subtract)
├─ Save wallet
└─ Return true (successfully undone)


CreditDestinationWalletStep:

execute():
├─ Lock wallet
├─ Add amount
├─ Save wallet
└─ Return true (success)

compensate():
├─ Lock wallet (same wallet)
├─ Subtract amount (reverse of add)
├─ Save wallet
└─ Return true (successfully undone)
```

The order of compensation is REVERSE of execution:
```
Execution order:   Step 1 → Step 2 → Step 3
Compensation order: Step 3 ← Step 2 ← Step 1

This ensures we undo dependencies in correct order!
```

## Understanding Database Sharding

### What is Sharding?

Sharding is splitting your data across multiple databases based on a "shard key". Instead of one big database, you have multiple smaller databases, each holding a slice of the data.

**Why?** As your app grows, one database can't handle all users. Sharding lets you scale horizontally (add more databases).

### How Sharding Works in This App

```
Shard Key: user_id (modulo 2)
Formula: user_id % 2 + 1 determines which shard

User ID 1: 1 % 2 = 1 → 1 + 1 = 2 → realshard2
User ID 2: 2 % 2 = 0 → 0 + 1 = 1 → realshard1
User ID 3: 3 % 2 = 1 → 1 + 1 = 2 → realshard2
User ID 4: 4 % 2 = 0 → 0 + 1 = 1 → realshard1
User ID 5: 5 % 2 = 1 → 1 + 1 = 2 → realshard2
...and so on
```

**Visual representation:**

```
┌─────────────────────────────────────────┐
│   Single Database (Before Sharding)     │
│                                         │
│   Users: 1,2,3,4,5,6,7,8,9,10...        │
│   Wallets: all 10+ wallets              │
│   Transactions: 1000+ records           │
│                                         │
│   Problem: Too much data! Slow queries! │
└─────────────────────────────────────────┘


After Sharding:

┌──────────────────────────┐  ┌──────────────────────────┐
│   realshard1             │  │   realshard2             │
│   (Even user IDs)        │  │   (Odd user IDs)         │
│                          │  │                          │
│  Users: 2,4,6,8,10...    │  │  Users: 1,3,5,7,9...     │
│  Wallets: 2,4,6,8,10...  │  │  Wallets: 1,3,5,7,9...   │
│  Transactions: 500 recs  │  │  Transactions: 500 recs  │
│                          │  │                          │
│  Size: ~5 GB             │  │  Size: ~5 GB             │
└──────────────────────────┘  └──────────────────────────┘

Benefits:
├─ Each database is smaller
├─ Queries are faster (less data to scan)
├─ Can scale: just add more shards when needed
└─ Each shard can be on different server
```

### How ShardingSphere Routes Queries

You write normal SQL, ShardingSphere transparently routes it:

```
Query: SELECT balance FROM wallet WHERE user_id = 5

Step 1: Parse SQL
        ShardingSphere reads: WHERE user_id = 5

Step 2: Calculate shard
        5 % 2 = 1 → 1 + 1 = 2
        Need shard: realshard2

Step 3: Route query
        Execute: SELECT balance FROM wallet WHERE user_id = 5
        Against: realshard2 database
        FROM: jdbc:mysql://localhost:3306/realshard2

Step 4: Return result
        ┌─────────────────┐
        │ balance: 1000   │
        └─────────────────┘
```

### Sharding in sharding.yml Configuration

```yaml
dataSources:
  realshard1:
    url: jdbc:mysql://localhost:3306/realshard1
  realshard2:
    url: jdbc:mysql://localhost:3306/realshard2

rules:
  - !SHARDING
    tables:
      user:
        actualDataNodes: realshard${1..2}.user
        databaseStrategy:
          standard:
            shardingColumn: id                    # Shard by user.id
            shardingAlgorithmName: db-inline      # Use inline algorithm
      
      wallet:
        actualDataNodes: realshard${1..2}.wallet
        databaseStrategy:
          standard:
            shardingColumn: user_id               # Shard by wallet.user_id
            shardingAlgorithmName: db-inline-user-id

    shardingAlgorithms:
      db-inline:
        type: INLINE
        props:
          algorithm-expression: realshard${id % 2 + 1}      # id % 2 + 1
      
      db-inline-user-id:
        type: INLINE
        props:
          algorithm-expression: realshard${user_id % 2 + 1} # user_id % 2 + 1
```

This config says:
- User table is sharded by `id` column
- Wallet table is sharded by `user_id` column
- Both use modulo 2 algorithm
- Data nodes span realshard1 and realshard2

### Why Multiple Shard Keys?

```
User table sharded by: id
  Why? Users are identified by their own ID
  User 1 → realshard2
  User 2 → realshard1

Wallet table sharded by: user_id
  Why? Each wallet belongs to one user
  Wallet of user 1 → realshard2 (same shard as user 1!)
  Wallet of user 2 → realshard1 (same shard as user 2!)

Result: Related data is on same shard = better performance!
```

### Sharding Benefits vs Drawbacks

**Benefits:**
```
✓ Horizontal scaling - add more shards
✓ Faster queries - less data to scan
✓ High availability - one shard down doesn't affect others
✓ Distributed load - queries spread across multiple servers
```

**Drawbacks (handled in this app):**
```
✗ Transactions across shards are complex (solved with saga pattern)
✗ Joins across shards are impossible (design tables per shard)
✗ Rebalancing is hard (need downtime to rehash)

In this app:
- When Alice (shard2) transfers to Bob (shard1):
  Saga handles it as multi-step instead of one transaction
- Each shard has all related tables (user, wallet, transaction)
- Joins only happen within a shard
```

## Project Structure

```
src/
├── config/
│   └── SagaConfiguration.java      # Registers saga steps
├── controller/
│   ├── UserController.java         # User CRUD
│   ├── WalletController.java       # Wallet operations
│   └── TransactionController.java  # Money transfers
├── models/
│   ├── User.java
│   ├── Wallet.java
│   ├── Transaction.java
│   ├── SagaInstance.java           # Tracks saga execution
│   ├── SagaStep.java               # Tracks each step
│   └── *Status.java                # Enums
├── services/
│   ├── UserService.java
│   ├── WalletService.java
│   ├── TransactionService.java
│   └── TransferSagaService.java    # Orchestrates saga
├── adapters/
│   └── *Adapter.java               # DTO converters
└── repository/
    └── *Repository.java            # Database access
```

## Technology Stack

| Component | Technology |
|-----------|-----------|
| Framework | Spring Boot 3 |
| Database | MySQL 8+ |
| Sharding | Apache ShardingSphere 5.x |
| Locking | JPA Pessimistic Locking |
| Pattern | Saga (Orchestration) |
| ORM | JPA/Hibernate |
| IDs | Snowflake |

## Saga Orchestration

### Saga Instance Lifecycle
```
STARTED → RUNNING → COMPLETED (success)
                 ↘ FAILED → COMPENSATING → COMPENSATED (rollback)
```

### Saga Step Lifecycle
```
PENDING → RUNNING → COMPLETED (success)
                 ↘ FAILED → COMPENSATING → COMPENSATED (rollback)
```

## Pessimistic Locking Detailed

### The Race Condition Problem

Without locking, money can disappear:

```
Initial state:
  Alice's wallet: balance = 100
  Bob's wallet: balance = 0

Transfer 1 (Alice → Bob: $50)
Transfer 2 (Alice → Bob: $60)

Timeline WITHOUT LOCK:

Time T1: Transfer 1 - Read Alice's balance
         Value read: 100 ✓

Time T2: Transfer 2 - Read Alice's balance
         Value read: 100 ✓ (Should be different but reads stale value!)

Time T3: Transfer 1 - Debit 50 from 100
         Alice = 50
         Save to DB

Time T4: Transfer 2 - Debit 60 from 100 (WRONG! Should be 50)
         Alice = 40
         Save to DB (overwrites previous save)

Result:  Alice = 40 (Should be -10, but we gave error)
         But the issue is: Bob gets credited twice
         
         Bob's balance:
         Transfer 1: 0 + 50 = 50
         Transfer 2: 50 + 60 = 110
         
TOTAL: Alice 40 + Bob 110 = 150 (Started with 100! Money created!)
```

### How Pessimistic Locking Fixes It

```
Timeline WITH PESSIMISTIC LOCK:

Time T1: Transfer 1 - Request LOCK on Alice's wallet
         ✓ LOCK ACQUIRED (Transfer 1 owns it)
         Read balance: 100

Time T2: Transfer 2 - Request LOCK on Alice's wallet
         ⏳ WAITING (Transfer 1 still has lock)

Time T3: Transfer 1 - Debit 50 from 100
         Alice = 50
         Save to DB

Time T4: Transfer 1 - RELEASE LOCK
         Transfer 2 can now PROCEED

Time T5: Transfer 2 - LOCK ACQUIRED
         Read balance: 50 (NOW CORRECT!)
         Try to debit 60
         ✗ ERROR: Insufficient balance (50 < 60)

Result:  Alice = 50 ✓
         Bob = 50 ✓
         Transfer 2 rejected ✓
         
TOTAL: Alice 50 + Bob 50 = 100 ✓ (Money preserved!)
```

### How It's Implemented in Code

```java
// In WalletRepository.java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT w FROM Wallet w WHERE w.userId = :userId")
Optional<Wallet> findByUserIdWithLock(@Param("userId") Long userId);

// In DebitSourceWalletStep.java
@Transactional
public boolean execute(SagaContext sagaContext) {
    Long fromUserId = sagaContext.getLong("fromUserId");
    BigDecimal amount = sagaContext.getBigDecimal("amount");
    
    // This query LOCKS the wallet
    Wallet wallet = walletRepository.findByUserIdWithLock(fromUserId)
            .orElseThrow(() -> new RuntimeException("Wallet not found"));
    
    // At this point, wallet is LOCKED exclusively
    // No other transaction can read or write this wallet
    
    log.info("Wallet Fetched with balance {}", wallet.getBalance());
    sagaContext.put("originalSourceWalletBalance", wallet.getBalance());
    
    wallet.debit(amount);  // Deduct the amount
    walletRepository.save(wallet);  // Save with lock still held
    
    // Lock is released when @Transactional method ends
    return true;
}
```

### Lock Types in JPA

```
PESSIMISTIC_READ:
├─ Allows other transactions to read
├─ Prevents other transactions from writing
└─ Use for: Reading data you need to be consistent

PESSIMISTIC_WRITE: (used in this app)
├─ Prevents ALL other access (read or write)
├─ Most restrictive but safest
└─ Use for: Modifying data (prevent any interference)

PESSIMISTIC_FORCE_INCREMENT:
├─ Like PESSIMISTIC_WRITE but also increments version
└─ Use for: Optimistic locking + pessimistic fallback
```

### Lock Duration

```
┌──────────────────────────────────────────────────────┐
│ @Transactional                                       │
│ public boolean execute(SagaContext sagaContext) {    │
│                                                      │
│   // LOCK ACQUIRED HERE                              │
│   Wallet wallet = findByUserIdWithLock(userId);      │
│                                                      │
│   wallet.debit(amount);                              │
│   walletRepository.save(wallet);                     │
│                                                      │
│   // LOCK RELEASED HERE (method ends)                │
│ }                                                    │
└──────────────────────────────────────────────────────┘

Lock lifetime: Duration of entire @Transactional method
(Called "READ COMMITTED" isolation level)
```

## Testing - Successful Transfer

```bash
# 1. Create users
curl -X POST http://localhost:8080/api/v1/user \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice","email":"alice@example.com"}'

# Response: { "id": 1, ... }

curl -X POST http://localhost:8080/api/v1/user \
  -H "Content-Type: application/json" \
  -d '{"name":"Bob","email":"bob@example.com"}'

# Response: { "id": 2, ... }

# 2. Create wallets
curl -X POST http://localhost:8080/api/v1/wallet \
  -H "Content-Type: application/json" \
  -d '{"userId":1}'

curl -X POST http://localhost:8080/api/v1/wallet \
  -H "Content-Type: application/json" \
  -d '{"userId":2}'

# 3. Add $500 to Alice
curl -X POST http://localhost:8080/api/v1/wallet/user/1/credit \
  -H "Content-Type: application/json" \
  -d '{"amount":500.00}'

# 4. Check Alice balance (should be $500)
curl http://localhost:8080/api/v1/wallet/user/1/balance

# 5. Transfer $100 Alice → Bob
curl -X POST http://localhost:8080/api/v1/transaction \
  -H "Content-Type: application/json" \
  -d '{
    "fromUserId":1,
    "toUserId":2,
    "amount":100.00,
    "description":"Coffee money"
  }'

# 6. Check balances
curl http://localhost:8080/api/v1/wallet/user/1/balance  # $400
curl http://localhost:8080/api/v1/wallet/user/2/balance  # $100
```

## Sharding Explanation

Why? Split data across databases for horizontal scaling.

How? ShardingSphere determines shard key and routes queries:
- User 1 → realshard1
- User 2 → realshard2  
- User 3 → realshard1
- User 4 → realshard2

You write normal queries, sharding is transparent!

## Common Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| Wallet not found | User doesn't exist | Create user first |
| Insufficient balance | Not enough money | Use /credit endpoint |
| Cannot transfer to same user | fromUserId == toUserId | Use different users |
| Sharding error | Database missing | Create realshard1 and realshard2 |
5. **Orchestration** - Coordinate workflow steps

