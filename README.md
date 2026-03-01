# Wallet App - Money Transfer System

A Spring Boot application for handling money transfers between users with saga pattern for distributed transactions. Includes database sharding for scalability.

## What This Does

Users can transfer money from one wallet to another. The system uses a saga pattern to handle multi-step transactions reliably, even if something fails mid-transfer.

**Example**: User A transfers $100 to User B
1. Deduct $100 from User A's wallet
2. Add $100 to User B's wallet  
3. Mark transaction as SUCCESS
4. If any step fails → rollback all previous steps

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

2. **Run Application**
```bash
mvn clean install
mvn spring-boot:run
```

App runs on `http://localhost:8080`

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

## How It Works

### Transfer Flow

```
User A (balance: $100) → Transfer $50 → User B (balance: $50)

Step 1: Debit User A's Wallet
  - Lock User A's wallet
  - Check balance ($100 >= $50 ✓)
  - Subtract $50
  - User A now has $50
  ✓ COMPLETED

Step 2: Credit User B's Wallet
  - Lock User B's wallet
  - Add $50
  - User B now has $100
  ✓ COMPLETED

Step 3: Update Transaction Status
  - Mark transaction as SUCCESS
  ✓ COMPLETED

Transaction Successful ✓
```

### If Something Fails

```
Step 1: Debit User A's Wallet ✓ COMPLETED
Step 2: Credit User B's Wallet ✓ COMPLETED
Step 3: Update Transaction Status ✗ FAILED (e.g., DB connection lost)

→ Compensation starts (ROLLBACK):

Compensate Step 2:
  - Lock User B's wallet
  - Subtract $50 (undo the credit)
  ✓ COMPENSATED

Compensate Step 1:
  - Lock User A's wallet
  - Add $50 back (undo the debit)
  ✓ COMPENSATED

Transaction Cancelled - Both users back to original balance ✓
```

## Key Features

### 1. Saga Pattern
- Multi-step transaction handling
- Automatic rollback if any step fails
- Each step is independent

### 2. Database Sharding
Data split across 2 databases based on rules:
- User table: Sharded by `id` (user 1,3,5... → realshard1, user 2,4,6... → realshard2)
- Wallet table: Sharded by `user_id` (same rule)
- Transaction table: Sharded by `id`

### 3. Pessimistic Locking
Wallets are locked during transactions to prevent race conditions

### 4. Saga Context
Shared data passed between steps with balance snapshots

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

## Concurrency Control

Problem: Two transfers to same user at same time - without locking both could read stale balance and both would succeed incorrectly.

Solution: Pessimistic locking locks the wallet during transaction so only one transfer can access it at a time.

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

## What You Learned

1. **Saga Pattern** - Multi-step distributed transactions
2. **Database Sharding** - Horizontal scaling with ShardingSphere
3. **Pessimistic Locking** - Prevent race conditions
4. **Transaction Rollback** - Compensating transactions
5. **Orchestration** - Coordinate workflow steps
