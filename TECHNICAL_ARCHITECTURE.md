# ⚙️ Technical Architecture & Workflows

### ◆ User Flow (Flow Chart)
This flow demonstrates the **Financial Escrow Logic** integrated with the user journey.

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
        Deduct -- Insufficient --> Fail[❌ Error: Low Funds]
        Deduct -- Sufficient --> Lock[💸 Deduct from Provider & LOCK in Escrow]
        
        Lock --> Chat[💬 Open Chat Channel]
        Chat --> Execute[Task Execution]
        Execute --> OTP[OTP Verification]
        
        %% Financial Step 2
        OTP --> Release[💰 Unlock Funds & Transfer to Hero]
        Release --> Finish((Task Complete))
    end
