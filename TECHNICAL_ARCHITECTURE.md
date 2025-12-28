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
    participant TM as Task Master
    participant API as Node.js API
    participant DB as MongoDB
    participant H as Hero

    Note over TM, DB: Phase 1: Creation (Money moves to Safe)
    TM->>API: POST /create-task (Price: X)
    API->>DB: Check Master Balance
    
    alt Sufficient Funds
        DB->>DB: $inc: { wallet: -X } (Debit Master)
        DB->>DB: Insert Task { status: "OPEN", escrow_amt: X }
        API-->>TM: "Quest Posted & Funds Locked"
    else Low Balance
        API-->>TM: Error: Recharge First
    end

    Note over H, DB: Phase 2: The Agreement
    H->>API: GET /feed
    H->>API: POST /accept-task
    API->>DB: Update Task { status: "IN_PROGRESS", hero: H_ID }
    API-->>H: "Quest Accepted!"
    API-->>TM: "Hero found!"

    rect rgb(30, 30, 30)
        Note right of API: ESCROW STATE: Money is held in Task Doc
        TM->>H: Chat: "Bring to Block A"
        H->>TM: Chat: "Arrived, give OTP"
    end

    Note over H, DB: Phase 3: Completion (Money moves to Hero)
    H->>API: POST /verify-otp (Code: 1234)
    API->>DB: Validate OTP
    
    alt OTP Valid
        DB->>DB: Update Task { status: "COMPLETED", escrow_amt: 0 }
        DB->>DB: $inc: { wallet: +X } (Credit Hero)
        API-->>H: "Payment Released!"
        API-->>TM: "Quest Completed"
    else Invalid OTP
        API-->>H: Error: Incorrect OTP!
    end
```
### ◆ System Architecture Diagram (High-Level)
This structural diagram shows how our Tech Stack components interact. We follow a standard **Client-Server Architecture**.

```mermaid
graph TD
    subgraph "Client Side (Frontend)"
        User[ Student User]
        UI[ React + Vite App]
        User -->|Interacts| UI
    end
```
```mermaid
graph TD
    subgraph "Server Side (Backend)"
        API[ Node.js + Express API]
        Auth[ Auth Middleware]
        Logic[ Escrow Logic]
        
        UI -->|HTTP Requests: Axios| API
        API --> Auth
        API --> Logic
    end
```
```mermaid
graph TD
    subgraph "Data Layer"
        DB[( MongoDB)]
        
        Logic -->|Mongoose Queries| DB
        Auth -->|Find User Doc| DB
    end
```
