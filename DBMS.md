# DBMS — Day 3

## ACID Properties

- **Atomicity:** All operations in a transaction succeed completely or 
  none are applied. On failure, MySQL rolls back using the InnoDB undo 
  log — restoring the database to its exact state before the 
  transaction began. There is no automatic restart.

- **Consistency:** The database moves from one valid state to another 
  valid state, honoring all constraints, foreign keys, and rules. 
  Consistency is guaranteed because Atomicity did its job.

- **Isolation:** Each transaction executes as if it is the only 
  transaction in the system. Intermediate states are hidden from 
  concurrent transactions. Without isolation, two users can read the 
  same seat as available simultaneously and both book it — this is the 
  lost update problem.

- **Durability:** Committed data survives system crashes. MySQL 
  achieves this via write-ahead logging — changes are written to disk 
  log before being applied to the database.

## Indexing

- **Definition:** A separate data structure MySQL maintains alongside 
  a table to speed up row retrieval without scanning every row.

- **Internal data structure:** B-Tree (default in MySQL InnoDB).

- **How B-Tree works:** Stores indexed column values in sorted order. 
  Each leaf node points to the actual row location on disk. Query 
  traverses the tree in O(log n) time.

- **Time complexity with index:** O(log n)

- **Time complexity without index:** O(n) — full table scan

- **BookMyShow usage:** Index on `show_id` column on the `bookings` 
  table. Every seat availability check queries bookings by show_id. 
  Without this index, MySQL scans every booking row in the table. 
  With it, MySQL jumps directly to matching rows in O(log n).

## Transactions

- **Definition:** A set of database operations that execute as a 
  single atomic unit. Either all operations commit or none do.

- **Rollback mechanism:** InnoDB undo log. Before writing any change 
  to disk, MySQL records original values in the undo log. On failure, 
  MySQL reads the undo log and reverses every completed operation.

- **BookMyShow booking transaction — three operations in order:**
  1. Mark seat as booked
  2. Deduct payment
  3. Write booking row

- **If payment succeeds but booking row write fails:** Atomicity 
  triggers rollback. InnoDB undo log reverses the payment deduction 
  and the seat status returns to available. User sees failure message. 
  Application layer decides whether to retry. MySQL never retries 
  automatically.

## Locking

- **Pessimistic locking definition:** Assumes conflicts are frequent. 
  Acquires exclusive lock on the row before reading it. No other 
  transaction can touch that row until lock is released.

- **Pessimistic locking MySQL command:** `SELECT * FROM seats WHERE 
  id = ? FOR UPDATE`

- **Optimistic locking definition:** Assumes conflicts are rare. Reads 
  without locking. Uses a version column to detect conflicts at commit 
  time. If version changed since read, transaction is rejected.

- **Optimistic locking mechanism:** Row has a `version` column. 
  Transaction reads row at version 5. Before committing, checks if 
  version is still 5. If another transaction changed it to 6, this 
  transaction is rejected with a version mismatch error.

- **What BookMyShow uses and why:** Redis SET NX — pessimistic 
  behavior. Seat is claimed exclusively at the start before any other 
  request can proceed. Under 500 concurrent users, conflicts on 
  popular seats are guaranteed, not rare. Catching conflicts at commit 
  time via optimistic locking is too late — payment processing has 
  already begun. Redis SET NX prevents two users from ever reaching 
  payment simultaneously for the same seat.

- **Redis SET NX exact command:**
  `SET seat:lock:{showId}:{seatId} {userId} NX EX 300`
  - NX — only set if key does not exist
  - EX 300 — expire in 300 seconds
  - Returns OK if lock acquired, nil if seat already locked
