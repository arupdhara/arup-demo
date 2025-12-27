### ◆ User Journey
From posting a task to getting paid.

```mermaid
graph TD
    Login[Login] --> Home[Quest page]
    Home --> Post[Post Quest]
    Post --> Escrow[Funds held in Escrow]
    Escrow --> Hero[Hero Accepts Quest]
    Hero --> OTP[OTP Verification]
    OTP --> Release[Payment Released to Hero's wallet]
```
### ◆ Technical Workflow
The step-by-step process and logic is shown.
```mermaid
graph TD
    Start((Start)) --> Auth{Has Account?}
    Auth -- No --> SignUp
    Auth -- Yes --> Login[Login]
    
    Login --> Dashboard[User Dashboard]
    
    subgraph "Marketplace Logic"
        Dashboard --> Post[Task Provider: Post Task]
        Dashboard --> Feed[Hero: Browse Feed]
        
        Post --> DB[(Database)]
        Feed --> Bid[Hero: Bid on Task]
        
        Bid --> Accept[Provider: Accept Bid]
        
        %% Financial Step 1
        Accept --> Deduct{Check Balance}
        Deduct -- Insufficient --> Fail[Error: Low Funds]
        Deduct -- Sufficient --> Lock[Deduct from Provider & LOCK in Escrow]
        
        Lock --> Chat[Open Chat Channel]
        Chat --> Execute[Task Execution]
        Execute --> OTP[OTP Verification]
        
        %% Financial Step 2
        OTP --> Release[Unlock Funds & Transfer to Hero]
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
        User[Student User]
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
