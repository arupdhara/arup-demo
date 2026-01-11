### ◆ User Journey
From posting a task to the secure OTP handshake.

```mermaid
graph TD
    Start((Start)) --> Auth{New User?}
    
    Auth -- Yes (create account) --> Register[Enter Details]
    Register --> OTP_Gen[System Sends Email OTP]
    OTP_Gen --> Input_OTP[User Enters OTP]
    Input_OTP --> Verify_OTP{Valid?}
    
    Verify_OTP -- No --> Retry[Retry]
    Verify_OTP -- Yes --> Home[Post Page]
    
    Auth -- No --> Login[Login]
    Login --> Home
    Home --> RoleSelection{Choose Role}
    RoleSelection -- TaskMaster --> TM[Stay on Post Page]
    RoleSelection -- Hero --> Hero[Go to Find Page]
    
    %%Core Logic
    TM --> Post[Post Quest]
    Hero --> Find[View Feed]
    Post --> Escrow[Funds DEDUCTED & Locked]
    Escrow --> Feed[Quest appears on Feed]
    
    Find --> Decision{Direct Accept or Bid?}
    Decision -- Direct Accept --> Lock[Atomic Lock: First Click Wins]
    Decision -- Bid --> Bidding[Hero Places Bid]
    
    Bidding --> MasterDecision{Master Action?}
    
    MasterDecision -- Accepts Bid --> TMAccept[Master Accepts Bid]
    MasterDecision -- Ignores --> FeedRetained[Quest Stays on Feed]
    
    TMAccept -- Atomic Lock: First Click Wins --> Adjust[Wallet Auto-Adjusts]
    
    Adjust --> Working[Quest In Progress]
    Lock --> Working
    
    Working --> Outcome{Smooth or Conflict?}
    
    Outcome -- Smooth --> OTP[Hero meets Master -> Gets OTP]
    OTP --> Verify[OTP Verification]
    Verify --> Release[Payment Unlocked to Hero]
    
    Outcome -- Conflict --> Dispute[Raise Dispute]
    Dispute --> EmailBridge[Email Bridge Activated]
    EmailBridge --> Admin[Admin Resolves the isuue]
    Admin --> Refund[Refund to Task Master or Pay Hero or Split the money to both]
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
        
        %% Hero Flow & Atomic Locking
        Lock --> Feed[Live Global Feed]
        Dashboard -- Hero --> Feed
        Feed --> Action{Accept or Bid?}
        
        Action -- Bid --> PlaceBid[Add to Bid List]
        Action -- Accept --> Atomic{Is Status == 'OPEN'?}
        
        Atomic -- No --> RaceFail[Error: Too Late!]
        Atomic -- Yes --> DB_Lock[Update: Status='ACTIVE']
        
        %% Completion Flow
        DB_Lock --> Execute[Execution Phase]
        Execute --> Result{User Action}
        
        %% Happy Path
        Result -- Verify OTP --> Success[Unlock Funds -> Credit Hero]
        
        %% Dispute Path
        Result -- Raise Dispute --> Bridge[Trigger Email Bridge]
        Bridge --> Admin[Admin Panel]
        Admin -- Resolve --> PaySplit[Execute Split/Refund]
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
    participant MAIL as Nodemailer

    Note over TM, DB: Phase 1: Creation & Bidding
    TM->>API: POST /create-task (Reward: ₹50)
    API->>DB: Debit ₹50 & Create { status: "OPEN" }
    
    alt Hero Bids Higher
        H->>API: POST /bid (Amount: ₹70)
        API->>DB: Push to Bids Array
        TM->>API: PUT /accept-bid
        API->>DB: Debit Extra ₹20 & Lock Task
    else Direct Accept (Race Condition Safe)
        H->>API: PUT /accept-task
        API->>DB: findOneAndUpdate({ _id: X, status: "OPEN" })
        alt Success (Lock Acquired)
             DB->>DB: Set AssignedTo = Hero
             API-->>H: "Quest Assigned!"
        else Fail (Already Taken)
             API-->>H: "Error: Quest no longer available"
        end
    end

    Note over H, DB: Phase 2: Execution & Disputes
    
    rect rgb(30, 30, 30)
        Note right of API: DISPUTE SCENARIO
        H->>API: POST /raise-dispute (Reason: "He ghosted me")
        API->>DB: Set Status="DISPUTED" (Lock Funds)
        API->>MAIL: Send Threaded Email (BCC Admin)
        MAIL-->>TM: "Reply with Proof"
        MAIL-->>H: "Reply with Proof"
        Note right of MAIL: Privacy: Emails are masked
    end

    Note over H, DB: Phase 3: Resolution
    
    alt Happy Path (OTP)
        H->>API: POST /verify-otp
        DB->>DB: Credit Hero & Close Task
    else Admin Resolution
        API->>DB: POST /resolve (Split 50/50)
        DB->>DB: Credit TM ₹35, Credit Hero ₹35
        DB->>DB: Close Task
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
        
        subgraph "Core Engines"
            Logic[Escrow & Rounding Logic]
            Race[Atomic Lock Manager]
            Dispute[Dispute Resolution Center]
        end
        
        UI -->|HTTP Requests| API
        API --> Auth
        API --> Logic
        API --> Race
        API --> Dispute
    end

    subgraph "External Services"
        Mail[Gmail SMTP Nodemailer]
        Dispute -->|Async Email| Mail
    end

    subgraph "Data Layer"
        DB[(🍃 MongoDB)]
        
        Logic -->|Transactions| DB
        Race -->|findOneAndUpdate| DB
    end
