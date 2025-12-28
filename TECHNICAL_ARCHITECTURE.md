#  Technical Architecture

### ◆ User Journey
From posting a task to the secure OTP handshake.

```mermaid
graph TD
    Login[Login] --> Home[Task Master page]
    Home --> Post[Post Quest]
    Post --> Escrow[Funds DEDUCTED & Locked]
    Escrow --> Feed[Quest appears on Feed]
    Feed --> Hero[Hero Accepts Quest]
    Hero --> Working[Quest In Progress]
    Working --> OTP[Hero meets Master -> Gets OTP]
    OTP --> Verify[OTP Verification]
    Verify --> Release[Payment Unlocked to Hero]
```
### ◆ Technical Workflow
The step-by-step process and logic is shown.
```mermaid
graph TD
    Start((Start)) --> Auth{Has Account?}
    Auth -- No --> SignUp
    Auth -- Yes --> Login[Login]

    subgraph "Marketplace Logic"
        Login --> Dashboard[Select Role]
        SignUp --> Dashboard 
        
        
        %% Posting Flow
        Dashboard -- Task Master --> Post[Post Task]
        Post --> Check{Balance > Price?}
        Check -- No --> Fail[Error: Insufficient Funds]
        Check -- Yes --> Lock[Debit Wallet -> Create Task]
        
        %% Hero Flow
        Lock --> Feed[Live Global Feed]
        Dashboard -- Hero --> Feed
        Feed --> Accept[Accept Quest]
        Accept --> Remove[Remove from Feed -> Assign to Hero]
        
        %% Completion Flow
        Remove --> Execute[Execution Phase]
        Execute --> Verify{Enter OTP}
        
        Verify -- Valid --> Success[Unlock Funds -> Credit Hero]
        Verify -- Invalid --> Retry[Error: Incorrect OTP!]
        
        %% Auto-Refund Logic
        Execute -- Timeout --> Refund[Auto-Refund to Task Master]
    end
```
### ◆ Data Flow Diagram (DFD)
This sequence diagram details the Database Transactions. We use "Double-Entry" logic (Deduct first, Hold, then Release) to ensure money is never lost.
```mermaid
sequenceDiagram
    participant TM as Task Master
    participant API as Node.js API
    participant DB as MongoDB
    participant H as Hero

    Note over TM, DB: Phase 1: Creation (Money moves to Safe)
    TM->>API: POST /create-task (Price: X)
    API->>DB: Check Master Balance
    
    alt Sufficient Funds
        DB->>DB: $inc: { wallet: -X } (Debit Master)
        DB->>DB: Insert Task { status: "OPEN", escrow_amt: X, visibility: "PUBLIC" }
        API-->>TM: "Quest Posted!" Visible on Dashboard & Feed
    else Low Balance
        API-->>TM: Error: Recharge First
    end

    Note over H, DB: Phase 2: The Race (Feed Logic)
    H->>API: GET /feed (Shows only OPEN tasks)
    H->>API: POST /accept-task
    
    Note right of DB: STATE CHANGE: Task moves from Public to Private
    DB->>DB: Update Task { status: "IN_PROGRESS", visibility: "PRIVATE" }
    API-->>H: "Quest Assigned!"
    API-->>TM: "Hero found! Check Dashboard."

    rect rgb(30, 30, 30)
        Note right of API: ESCROW VAULT (Money Locked)
        TM->>H: Chat: "I am at the Library."
        H->>TM: Chat: "I'm here. Code?"
        TM->>H: (Verbal): "Code is 4592"
    end

    Note over H, DB: Phase 3: Completion OR Refund
    
    alt Hero Enters Valid OTP
        H->>API: POST /verify-otp
        DB->>DB: Update { status: "COMPLETED", escrow_amt: 0 }
        DB->>DB: $inc: { wallet: +X } (Credit Hero)
        API-->>H: "Payment Received!"
        API-->>TM: "Quest Closed"
    else Hero Enters wrong OTP
        API-->>H: Error: Incorrect OTP!
    else Deadline Passed (Auto-Refund)
        Note right of API: TTL Expired (No OTP entered)
        DB->>DB: Update { status: "EXPIRED", escrow_amt: 0 }
        DB->>DB: $inc: { wallet: +X } (Refund Master)
        API-->>TM: "Task Expired. Money Refunded."
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

    subgraph "Server Side (Backend)"
        API[Node.js + Express API]
        Auth[Auth Middleware]
        Logic[Escrow & OTP Logic]
        
        UI -->|HTTP Requests| API
        API --> Auth
        API --> Logic
    end

    subgraph "Data Layer"
        DB[(🍃 MongoDB)]
        
        Logic -->|Mongoose Queries| DB
        Auth -->|Find User Doc| DB
    end
```
