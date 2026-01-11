## ◆ User Journey
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
    Dispute --> EmailBridge[Email Bridge Activated means mail received by both hero and master, they send proof and details through mail]
    EmailBridge --> Admin[Admin Resolves the isuue]
    Admin --> Refund[Refund to Task Master or Pay Hero or Split the money to both]
```
## ◆ Technical Workflow
The step-by-step process and logic is shown.
```mermaid
graph TD
    subgraph "Phase 1: Authentication & Security"
        Start((Init)) --> CheckSession{Token Exists?}
        CheckSession -- Yes --> AutoLogin[Restore Session]
        CheckSession -- No --> AuthScreen
        
        AuthScreen --> Action{Create Account or Login?}
        
        Action -- Login --> Creds[Verify Email/Pass]
        Creds -- Success --> LoadUser
        Creds -- Fail --> ErrorMsg
        
        Action -- Create Account --> Validate[Domain/Age Check]
        Validate -- Pass --> GenOTP[Generate 6-Digit Code]
        GenOTP --> SendMail[Nodemailer: Send to Inbox]
        
        SendMail --> Wait[Wait for OTP Input]
        Wait --> Verify{Code Matches?}
        
        Verify -- No --> Reject[Block Account Creation]
        Verify -- Yes --> CreateDB[Insert User into MongoDB]
        CreateDB --> LoadUser
    end
    
    subgraph "Phase 2: Marketplace Logic"
        LoadUser --> Dashboard[Select Role]
        
        %% Posting Flow
        Dashboard -- Task Master --> Post[Post Task]
        Post --> Check{Balance > Price?}
        Check -- No --> Fail[Error: Insufficient Funds]
        Check -- Yes --> Lock[Debit Price from Wallet -> Create Task]
        
        %% Hero Flow & Atomic Locking
        Lock --> Feed[Live Global Feed]
        Dashboard -- Hero --> Find[Find Quest]
        Find --> Decision{Accept or Bid?}
        
        %% BIDDING PATH (Connected to Master)
        Decision -- Bid --> PlaceBid[Add to Bid List]
        PlaceBid --> MasterCheck{Master Accepts?}
        
        MasterCheck -- No --> Feed[Remains on Feed]
        MasterCheck -- Yes --> BidLock[Debit difference of Bid & Price from Wallet -> TaskStatus='ACTIVE']
        
        %% DIRECT ACCEPT PATH
        Decision -- Accept --> Atomic{Is TaskStatus == 'OPEN'?}
        
        Atomic -- No --> RaceFail[Error: Too Late!]
        Atomic -- Yes --> DB_Lock[Update: TaskStatus='ACTIVE']
        
        %% MERGE POINTS: Both paths lead to Execution
        BidLock --> Execute[Execution Phase]
        DB_Lock --> Execute
        
        %% Completion Flow
        Execute --> Result{User Action}
        
        %% Happy Path
        Result -- Verify OTP --> Success[Unlock Funds -> Credit Hero]
        
        %% Dispute Path
        Result -- Raise Dispute --> Bridge[Trigger Email Bridge]
        Bridge --> Admin[Admin Panel]
        Admin -- Resolve --> PaySplit[Execute Split/Refund Master/Credit Hero]
    end
```
## ◆ Data Flow Diagram (DFD)
Shapes used:
* **External Entities [ ]**: Rectangles represent users (Student, Admin) or external systems (Gmail SMTP).

* **Processes ( )**: Rounded edge rectangles denote logical transformations (e.g., Auth Engine, Wallet Logic).

* **Data Stores [( )]**: Cylinders represent persistent storage (MongoDB Collections).

* **Data Flow -->**: Arrows indicate the movement of data, not the flow of control.

  
### DFD Level 0: Context Diagram
```mermaid
graph LR
    %% Entities
    Student[User]
    Admin[Admin]
    
    %% The System Boundary
    System((CampusJugaad System))
    
    %% Data Flows
    Student -- Credentials/OTP --> System
    Student -- Quest Details/Bids --> System
    Student -- Dispute Evidence --> System
    
    System -- Quest Status/Notifications --> Student
    System -- Wallet Balance Update --> Student
    
    Admin -- Resolution Decision --> System
    System -- Dispute Alerts --> Admin
```
### DFD Level 1: System Overview
```mermaid
graph TD
    %% Entities
    User[Student User]
    Admin[Admin]

    %% Data Stores (Open Rectangles)
    DS_User[(User Profile & Wallet)]
    DS_Quest[(Quest Repository)]
    DS_Txn[(Transaction Ledger)]

    %% Processes (Rounded Rectangles)
    P1(1.0 Authentication)
    P2(2.0 Quest Management)
    P3(3.0 Wallet Engine)
    P4(4.0 Dispute Resolution)

    %% Flow: Auth
    User -->|Login/Reg Info| P1
    P1 <-->|Verify Creds| DS_User
    P1 -->|Session Token| User

    %% Flow: Quests & Verification (UPDATED)
    User -->|1. Post/Accept/Bid| P2
    User -->|2. Submit OTP Code| P2
    P2 <-->|Read/Write Quest State| DS_Quest
    P2 -->|3. Verification Status Success/Fail| User
    
    %% Flow: Financials (Triggered by Quest)
    P2 -->|Request Fund Lock/Release| P3
    P3 <-->|Update Balance| DS_User
    P3 -->|Log Audit Trail| DS_Txn
    P3 -->|Confirmation| P2

    %% Flow: Disputes
    User -->|Raise Issue| P4
    P4 -->|Lock Quest| DS_Quest
    P4 -->|Freeze Funds| P3
    Admin -->|Settlement Instruction| P4
```
### DFD Level 2: The "Core" Process (Quest & Financials)
```mermaid
graph TD
    %% External Entity
    TM[Task Master]
    Hero[Hero]

    %% Data Stores
    Store_User[(User DB)]
    Store_Quest[(Quest DB)]
    Store_Ledger[(Ledger DB)]

    %% Sub-Processes
    P2_1(2.1 Validate Balance)
    P2_2(2.2 Lock Escrow)
    P2_3(2.3 Publish Quest)
    P2_4(2.4 Process Bid/Accept)
    P2_5(2.5 Verify OTP)
    P2_6(2.6 Settlement)

    %% Logic Flow
    TM -->|1. Quest Info + Reward| P2_1
    P2_1 <-->|Read Balance| Store_User
    
    P2_1 -->|Valid Funds| P2_2
    P2_2 -->|Debit Amount| Store_User
    P2_2 -->|Log Debit| Store_Ledger
    
    P2_2 -->|Funds Secured| P2_3
    P2_3 -->|Save New Quest| Store_Quest
    
    Hero -->|2. Bid or Accept| P2_4
    P2_4 <-->|Update Status 'Active'| Store_Quest
    P2_4 -->|Trigger Top-up if Bid > Reward| P2_2

    TM -->|3. Provide OTP| Hero
    Hero -->|4. Input OTP| P2_5
    
    P2_5 <-->|Validate OTP| Store_Quest
    P2_5 -->|OTP Valid| P2_6
    
    P2_6 -->|Credit Funds| Store_User
    P2_6 -->|Log Credit| Store_Ledger
    P2_6 -->|Close Quest| Store_Quest
```
## ◆ Backend Logic & Data Integrity
```mermaid
sequenceDiagram
    autonumber
    participant TM as Task Master
    participant H as Hero
    participant API as Node.js API
    participant DB as MongoDB (Atlas)

    Note over TM, DB: PHASE 1: POSTING (Debit & Lock)
    
    TM->>API: POST /quests (Reward: ₹100)
    
    rect rgb(30, 30, 30)
        Note right of API: 🛑 1. ATOMIC CHECK
        API->>DB: User.findOne({ _id: TM, balance: { $gte: 100 } })
        
        alt Insufficient Funds
            DB-->>API: null
            API-->>TM: Error: "Recharge Wallet!"
        else Balance Available
            Note right of API: 🛑 2. DOUBLE-ENTRY DEBIT
            API->>DB: User.updateOne({ $inc: { balance: -100 } })
            API->>DB: Quest.create({ status: 'OPEN', escrow: 100 })
            API->>DB: Transaction.create({ type: 'debit', amount: 100 })
            
            DB-->>API: Success
            API-->>TM: "Quest Posted & Funds Locked"
        end
    end

    Note over H, DB: PHASE 2: ATOMIC LOCKING (The "Race" Logic)

    H->>API: POST /bid (Offer: ₹120)
    API-->>TM: Notify: "New Bid: ₹120"
    
    TM->>API: PUT /accept-bid (Hero: H)
    
    rect rgb(30, 30, 30)
        Note right of API: 🛑 3. WALLET ADJUSTMENT
        Note right of API: Diff = 120 - 100 = 20
        API->>DB: User.updateOne({ $inc: { balance: -20 } })
        
        Note right of API: 🛑 4. CRITICAL SECTION (The Lock)
        API->>DB: Quest.findOneAndUpdate({ _id: Q, status: 'OPEN' }, { status: 'ACTIVE' })
        
        alt Lock Acquired
            DB-->>API: Updated Doc
            API-->>H: "Bid Won! Quest Assigned."
        else Lock Failed (Race Condition)
            DB-->>API: null
            API-->>TM: Error: "Quest already active!"
        end
    end

    Note over H, DB: PHASE 3: SETTLEMENT & DISPUTE
    
    H->>API: POST /verify-otp (OTP: "4592")
    
    rect rgb(30, 30, 30)
        Note right of API: 🛑 5. VERIFICATION
        API->>DB: Quest.findOne({ otp: "4592" })
        
        alt Valid OTP
            API->>DB: User.updateOne({ _id: H }, { $inc: { balance: +120 } })
            API->>DB: Quest.updateOne({ status: 'COMPLETED' })
            API-->>H: "₹120 Credited!"
        else Invalid
            API-->>H: Error: "Wrong OTP"
        end
    end
```
## ◆ System Architecture Diagram (High-Level)
This structural diagram shows how our Tech Stack components interact. We follow a standard **Client-Server Architecture**.

```mermaid
graph TD
    subgraph "Client Side (Frontend)"
        User[User]
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
