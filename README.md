# 🎓 Campus Jugaad: The Hyper-Local Campus Marketplace

**"Summon a Hero. Delegate your chaos."**

### ◆ Problem Statement
University campuses are isolated economies with inefficient "favor" systems. Students struggle to find help for small tasks like- 
a professor urgently needs a specific medicine but cannot leave the department because of back-to-back labs, a student needs a cycle for 1 hour, printing, food delivery, laundry pickup and many more.  
Right now, students randomly message WhatsApp groups, call friends, or run across campus in the middle of a busy schedule. There is no organized system to request help, no way to find someone available immediately, and no trusted way to pay for such small tasks.
While others have free time but no way to monetize it. Existing platforms cannot enter hostels or classrooms.

### ◆ Proposed Solution
**Campus Jugaad** is a closed-loop, secure marketplace designed exclusively for university campuses. It transforms the informal economy into a tech-driven system where students can outsource tasks to verified peers ("Heroes") safely.

##  Key Features

###  The Gamified Experience (User-Facing)
* **Gamified Quest System:** Tasks aren't just chores; they are "Quests" with specific bounties, deadlines, and urgency levels (Low/Medium/Urgent).
* **Hero Feed:** A real-time marketplace where students can find, filter, and accept quests based on location and reward to become the campus "Hero."
* **Competitive Leaderboards:** Students compete for "Guild Wars" (Hostel vs. Hostel) dominance. Top performers on the leaderboard earn real-world campus perks.
* **Legendary Dashboard:** Users track their journey with total earnings, completion stats, and a level-based XP system.

###  Security & Architecture (The "Brain")
* **Closed-Campus Authentication:** Integration with university emails (e.g., `@nitdgp.ac.in`) to ensure 100% verified college member access.
* **Zero-Fraud Escrow Wallet:** Payments are held in a secure `Escrow State` within the MySQL database and released only when the provider shares a secret OTP.
* **In-App Hero Chat:** A private communication channel for Task Masters and Heroes to negotiate details without sharing personal phone numbers.

###  Design & Accessibility
* **Mobile-First Design:** Fully responsive layout optimized for on-the-go campus use between classes.
* **Glassmorphic UI:** A modern, sleek interface featuring fluid animations and full Dark/Light mode support.

###  Feasibility & Impact
* **Market Size:** ~5,000 students per average technical campus.
* **Economic Impact:** projected **₹50,000+** monthly transaction volume in pilot phase.
* **Time Saving:** Average student saves **~3 hours/week** by outsourcing logistics.

###  Tech Stack
* **Frontend:** React.js, Vite, Tailwind CSS (Glassmorphism UI).
* **Backend:** Node.js, Express.js.
* **Database:** MySQL (Relational Data for transactions).
* **Auth:** JWT / Custom Campus Verification.

---
*For detailed architecture and diagrams, see [TECHNICAL_ARCHITECTURE.md](./TECHNICAL_ARCHITECTURE.md)*
*For future plans, see [ROUND_2_ROADMAP.md](./ROUND_2_ROADMAP.md)*
