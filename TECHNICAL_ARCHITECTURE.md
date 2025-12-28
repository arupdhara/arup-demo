### ◆ User Journey
From posting a task to getting paid.

```mermaid
graph TD
    Login[Login] --> Home[Task Master page]
    Home --> Post[Post Quest]
    Post --> Escrow[Funds lock in Escrow]
    Escrow --> Hero[Hero Accepts Quest]
    Hero --> Done[Quest Accomplished]
    Done --> OTP[OTP Verification]
    OTP --> Release[Payment Released to Hero's wallet]
```
### ◆ Technical Workflow
The step-by-step process and logic is shown.
```mermaid
graph TD
    Start((Start)) --> Auth{Has Account?}
    Auth -- No --> SignUp
    Auth -- Yes --> Login[Login]

    SignUp --> Dashboard{Select Role}    
    Login --> Dashboard
    
    subgraph "Marketplace Logic"
        Dashboard -- Task Master --> Post[Post Task]
        Dashboard -- Hero --> Feed[Browse Feed]
        
        Post --> Deduct{Check Balance}
        Deduct -- Insufficient --> Fail[Error: Insufficient Funds]
        Deduct -- Sufficient --> Lock[Deduct funds from Task Master's wallet & LOCK in Escrow]
        Lock --> DB[(Database)]
        
        Feed --> Accept[Accept Quest]        
        Accept --> Chat[Open Chat Channel]
        Chat --> Execute[Task Execution]
        Execute --> OTP[OTP Verification]
        
        %% Financial Step 2
        OTP -- Matched --> Release[Unlock Funds & Transfer to Hero]
        OTP -- Mismatched --> Denied[Error: Incorrect OTP! Please try again]
        Release --> Finish((Task Complete))
    end
```
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
```
### ◆ System Architecture Diagram (High-Level)
This structural diagram shows how our Tech Stack components interact. We follow a standard **Client-Server Architecture**.

```mermaid
graph TD
    subgraph "Client Side (Frontend)"
        User[User]
        UI[React + Vite App]
        User -->|Interacts| UI
    end
```
```mermaid
graph TD
    subgraph "Server Side (Backend)"
        API[Node.js + Express API]
        Auth[Auth Middleware]
        Logic[Escrow Logic]
        
        UI -->|HTTP Requests| API
        API --> Auth
        API --> Logic
    end
```
```mermaid
graph TD
    subgraph "Data Layer (Storage)"
        DB[(MySQL Database)]
        
        Logic -->|SQL Queries| DB
        Auth -->|Verify Creds| DB
    end
```
### ◆ Architecture Explanation
*We utilize a Client-Server Architecture with strict financial handling:*

**Presentation Layer**: React.js handles the UI, Chat interface, and state management.

**Application Layer**: Node.js/Express acts as the middleware. Crucially, it handles ACID Transactions for the wallet.

*Why*: If the server crashes after deducting money from the Provider but before assigning the Hero, the database transaction rolls back, ensuring no money is ever "lost" in the system.

**Data Layer**: MySQL stores relational data (Users, Tasks, Chat Logs) and the Escrow State.

**Escrow Logic**: Money does not go directly from A to B. It moves Provider -> Escrow Table -> Hero.
