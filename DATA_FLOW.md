### ◆ Data Flow Diagram (DFD)
This sequence diagram details the Database Transactions. We use "Double-Entry" logic (Deduct first, Hold, then Release) to ensure money is never lost.
```mermaid
sequenceDiagram
    participant P as Provider (User A)
    participant API as Node.js Server
    participant DB as MySQL Database
    participant H as Hero (User B)

    Note over P, DB: Phase 1: Agreement & Escrow Lock
    P->>API: POST /accept-bid (BidID)
    API->>DB: BEGIN TRANSACTION
    DB->>DB: UPDATE users SET wallet = wallet - price WHERE id = Provider
    DB->>DB: UPDATE tasks SET status = 'IN_PROGRESS', escrow_amount = price
    DB-->>API: ✅ Transaction Committed
    API-->>P: "Bid Accepted. Funds Locked."
    API-->>H: Notification: "You are hired!"

    Note over P, H: Phase 2: Execution & Chat
    P->>H: Chat: "Please bring it to Block A"
    H->>P: "I am here. OTP?"
    P->>H: "OTP is 4590"

    Note over H, DB: Phase 3: Settlement
    H->>API: POST /verify-otp (OTP: 4590)
    API->>DB: CHECK secret_otp match?
    
    alt Match Successful
        API->>DB: BEGIN TRANSACTION
        DB->>DB: UPDATE tasks SET status = 'COMPLETED', escrow_amount = 0
        DB->>DB: UPDATE users SET wallet = wallet + price WHERE id = Hero
        DB-->>API: ✅ Transaction Committed
        API-->>H: "Payment Received!"
        API-->>P: "Task Closed."
    else Match Failed
        API-->>H: ❌ Error: Incorrect OTP
    end
