# Collections & Disputes Assistant (CDA) - Design Document

**Version:** 1.0.0  
**Date:** 2025-12-04  
**Author(s):** Martin Kecka, Gemini AI  
**Category:** Assistant (A)  
**Acronym:** CDA

---

## 1. Introduction & Purpose

The **Collections & Disputes Assistant (CDA)** is an AI-driven automation module designed to optimize the "Order-to-Cash" cycle by intelligently managing outbound dunning (debt collection) communications.

While the existing **Cash Application Assistant (CAA)** focuses on inbound payment processing, CDA focuses on the outbound solicitation of outstanding debts. Its primary goal is to replace rigid, "dumb" spamming logic with a context-aware strategy that treats clients differently based on their payment history, debt age, and recent communication sentiment.

### Key Objectives

* **Intelligent Dunning:** Move beyond simple "Day + 7" logic. Segment invoices into behavioral buckets ("Fresh", "Aging", "Relics") and apply appropriate pressure.
* **Credit Awareness:** Implement **"Tactical Silence"** protocols to prevent automated harassment of clients with negative balances (credits/refunds due), preventing customer confusion.
* **Sentiment-Driven Pausing:** Automatically halt dunning campaigns if a client replies with specific intents (e.g., "Dispute", "Bankruptcy", "Promise to Pay"), flagging them for human review instead.
* **The "Priority Watchlist" (The Naughty List):** Provide a dashboard identifying chronic late payers versus occasional slippage, allowing the Finance team to focus effort where it yields the highest cash recovery.

---

## 2. Architecture & Ecosystem Integration

CDA is architected as a standalone microservice to preserve the stability and context-window efficiency of the core Cash Application system. It shares data sources but operates on an independent lifecycle.


### 2.1. The "SERA Finance Suite" Pattern
CDA sits alongside CAA, sharing the same "Data Truth" provider (CMC) but serving a different directional flow.

1.  **CMD Connector (CMC):** The shared source of truth. CDA uses CMC to fetch **Open Invoices** (the target list) and **Client Contacts** (the recipients).
2.  **Correspondence Mail Ingestor (CMI):** A **NEW** reusable module (distinct from RMI) designed to ingest, parse, and analyze unstructured email bodies from `billing@gtt.net`.
3.  **PostgreSQL Database:**
    * **Reads:** `open_invoices` (via CMC sync).
    * **Owns:** `dunning_campaigns`, `client_interaction_logs`, `dispute_cases`.

---

## 3. Core Components

### 3.1. The Dunning Engine (`core/dunning_engine.py`)
This is the deterministic logic core (based on the prototype script). It runs daily to analyze the entire Open Invoice ledger and categorize items.

* **Bucketing Logic:**
    * **Future:** Due Date > Now. **Action:** Sleep.
    * **Fresh (0-30 Days):** "Gentle Nudge" logic.
    * **Aging (31-365 Days):** "Escalation" logic.
    * **Relics (>1 Year):** "Bad Debt/Write-off" logic.
* **Credit Logic:** Checks balance < 0. **Action:** Tactical Silence.
* **Grouping:** Groups multiple open invoices for the same client into a single "Statement of Account" summary to avoid sending 50 separate emails to the same person.

### 3.2. Correspondence Mail Ingestor (CMI)
* **Role:** The "Ears" of the system.
* **Source:** `billing@gtt.net` (IMAP/Graph API).

**Differentiation from RMI:**
* **RMI:** Looks for Attachments (PDF/Excel) and structured tables. Ignores body text.
* **CMI:** Ignores attachments. Reads Body Text.

**AI Analysis:** Uses a fast LLM (e.g., `gemma2:27b`) to classify inbound emails into intents:
* **PROMISE_TO_PAY:** "I will pay next week." $\rightarrow$ **Action:** Snooze Dunning for X days.
* **DISPUTE:** "This tax amount is wrong." $\rightarrow$ **Action:** Stop Dunning, Create Dispute Case, Alert Human.
* **ADMINISTRATIVE:** "Update our address." $\rightarrow$ **Action:** Forward to Master Data Team.
* **AUTO_REPLY:** "Out of office." $\rightarrow$ **Action:** Ignore.

### 3.3. The Dashboard (UI)
A React-based interface focusing on Cash Recovery Potential.

* **Campaign View:** Upcoming emails queued for today (Review/Approve/Send All).
* **The Watchlist:** A ranked list of clients by "DSO Impact" (Days Sales Outstanding).
* **Dispute Inbox:** A filtered view of replies that stopped the automation, requiring human eyes.

---

## 4. Reusability Strategy (The "Steal" List)
To build CDA rapidly (in "Martin time"), we will heavily reuse components from the existing Cash Application Assistant ecosystem.

| Component | Source | Adaptation for CDA |
| :--- | :--- | :--- |
| **Data Provider** | CMC (`cmd_connector`) | Use `GetOpenInvoices` (existing) and `GetClientContacts` (existing). No new connectors needed. |
| **Client Lookup** | CMC | Use `GetClientByEmail` to map inbound `billing@gtt.net` replies to Client IDs. |
| **UI Framework** | CAA Frontend | Reuse `StatusChip`, `DataTable` components, and the basic React/FastAPI skeleton. |
| **Auth/Identity** | SERA Core | Reuse standard header-based authentication. |
| **Email Ingestion** | RMI (Infrastructure) | Reuse the `IMAPClient` / `GraphClient` connection classes from RMI, but feed the raw data to CMI for text analysis instead of RMI for table extraction. |
| **Rule Engine** | CAA Core | Reuse the Strategy Pattern structure (`BaseRule`, `RuleEngine`), but implement new `DunningRule` logic. |

---

## 5. Process Flow: The "Smart Loop"



[Image of automated workflow diagram]


1.  **Sync (06:00 AM):** CDA calls `CMC.GetOpenInvoices` to refresh the ledger.
2.  **Ingest (Continuous):** CMI watches the inbox.
    * If a reply arrives from `client@example.com`, CMI links it to the Client ID.
    * AI analyzes intent.
    * If intent == **DISPUTE** or **PROMISE**, a `Stop-List` flag is set for that Client ID in the database.
3.  **Analyze (08:00 AM):** The Dunning Engine runs.
    * Calculates `DaysOverdue` for all items.
    * Checks the `Stop-List`.
    * Generates a queue of **Proposed Emails**.
4.  **Review (09:00 AM - 10:00 AM):**
    * Jess/Finance Team logs in.
    * Reviews the "Proposed Emails" queue.
    * Removes any specific items manually if needed.
5.  **Execution (10:00 AM):**
    * CDA sends the batch.
    * Updates `last_contacted_date` in the database.

---

## 6. Future Capabilities

* **Payment Prediction:** Use historical payment data (from CAA) to predict when a client usually pays (e.g., "They always pay on the 45th day"). If today is Day 40, skip dunning because they are likely processing it.
* **Tone Matching:** AI generates the email body dynamically based on the client's historical relationship (e.g., Polite for VIPs, Firm for chronic late payers).