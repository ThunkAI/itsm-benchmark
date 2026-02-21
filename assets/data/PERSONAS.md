Based on the **ShopFlow** architecture and common organizational archetypes, here are 5 distinct employee personas.

These are designed to test the agent's ability to handle different levels of technical detail, emotional urgency, and ambiguity.

### 1. "The Hair-on-Fire" Sales Lead

**Name:** **Sarah Jenkins**
**Role:** Key Account Manager (Enterprise)
**Technical Proficiency:** **Low**
**Vibe:** High anxiety, revenue-focused, prone to exaggeration. She doesn't care _why_ it's broken, she just knows a $50k deal is stalling.
**ShopFlow Context:** Frequently flags issues with the **Checkout Flow** or **Frontend**.

- **Ticket Style:**
- **Subject:** URGENT!!! CHECKOUT BROKEN FOR VIP CLIENT!!!
- **Description:** "I'm on a demo with [BigClient] and the page is spinning. This is embarrassing. We are going to lose this deal if it's not fixed in 5 minutes. IS THE SERVER DOWN?? Call me ASAP."
- **Key Challenge for Agent:** Filtering out the emotional noise to identify the actual technical symptom (e.g., high latency vs. 500 error) and de-escalating the severity if it’s a local issue.

### 2. "The Metrics-Watcher" Product Manager

**Name:** **David Chen**
**Role:** PM for Storefront Experience
**Technical Proficiency:** **Medium** (Knows SQL, reads dashboards, but can't fix code).
**Vibe:** Data-driven, specific, slightly passive-aggressive. He will link you to a dashboard you might not have access to.
**ShopFlow Context:** Obsessed with **Core Web Vitals**, **Frontend Latency**, and **Conversion Rates**.

- **Ticket Style:**
- **Subject:** Regression in P99 Latency on PDP (Product Detail Page)
- **Description:** "I noticed a 15% bump in latency on the `frontend-storefront` starting around 2:00 PM UTC. It seems correlated with the new recommendation widget. Can we investigate if `recommendation-api` is throttling? See the graph here [Link]."
- **Key Challenge for Agent:** Investigating a specific hypothesis that might be wrong (a "red herring") while acknowledging the valid data provided.

### 3. "The Overworked" Customer Support Lead

**Name:** **Maria Rodriguez**
**Role:** Tier 2 Support Lead
**Technical Proficiency:** **Low-Medium** (Good at pattern recognition, bad at root cause).
**Vibe:** Exhausted, empathetic to users, factual. She aggregates user reports into one ticket.
**ShopFlow Context:** Deals with **Login Failures** (`auth-core-service`) or **Order Status** issues (`order-processor`).

- **Ticket Style:**
- **Subject:** Multiple reports of 500 errors during login
- **Description:** "Hey team, we've had about 40 tickets in the last hour from users in the EU saying they can't log in. They are getting a 'Something went wrong' red banner. Resetting passwords isn't helping. Is `auth-core-service` healthy?"
- **Key Challenge for Agent:** Correlating user volume to actual error rates and recognizing a regional or widespread outage based on "human sensors."

### 4. "The Grumpy" Backend Developer

**Name:** **Alex "Root" Kovalenko**
**Role:** Senior Backend Engineer (Platform Team)
**Technical Proficiency:** **High**
**Vibe:** Brief, technical, assumes you know what he knows. Annoyed that he has to file a ticket instead of just fixing it (likely doesn't have permissions for Prod).
**ShopFlow Context:** Notices **Infrastructure** or **Database** anomalies (`primary-db`, `redis-cache`).

- **Ticket Style:**
- **Subject:** Redis memory fragmentation critical
- **Description:** "Redis cache for `product-catalog` is evicting keys. Fragmentation ratio is > 1.5. We need to flush or scale it before the weekend traffic hits. I don't have access to the AWS console. Pls fix."
- **Key Challenge for Agent:** Taking action on a highly technical, prescriptive request without just blindly obeying (verifying the metric first).

### 5. "The Paranoid" Security Analyst

**Name:** **Samira Gupta**
**Role:** InfoSec Analyst
**Technical Proficiency:** **High** (Security specific).
**Vibe:** Formal, urgent, secretive. Treats everything as a potential breach until proven otherwise.
**ShopFlow Context:** Flags **Anomalous Traffic**, **DDoS signatures**, or **Vulnerability Scans**.

- **Ticket Style:**
- **Subject:** SECURITY INCIDENT: Anomalous outbound traffic from `payment-processor`
- **Description:** "SOC detected unexpected outbound connection attempts from `payment-processor` to an unlisted IP range (192.0.2.x) on port 443. This behavior is inconsistent with the baseline. Requesting immediate isolation of the service and log analysis."
- **Key Challenge for Agent:** Recognizing a "Scenario 09" type event where standard troubleshooting (restart) is wrong and escalation is the only path.

---

### Suggested "Voice" Implementation

When generating tickets for the benchmark, you can use a `persona_id` field to modulate the `description` text:

- **Sarah:** Use ALL CAPS, exclamation marks, and phrases like "As soon as possible."
- **David:** Use numbers, percentages, and "correlated with."
- **Alex:** Use technical jargon ("eviction," "fragmentation," "OOM") and short sentences.
