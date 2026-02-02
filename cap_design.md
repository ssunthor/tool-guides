# CV Analysis Processor (CAP) - Design Document

**Version:** 2.5.1
**Date:** 2026-02-02
**Author(s):** Martin Kecka, Gemini AI
**Category:** Processor (P)
**Acronym:** CAP

---

## 1. Introduction & Purpose

The **CV Analysis Processor (CAP)** is a specialized AI-driven module designed to automate the initial screening of candidate resumes. Its primary function is to provide hiring managers with a rapid, consistent, and in-depth analysis of a candidate's suitability for a given role by comparing their CV against the job description.

The CAP operates on a multi-pass analysis pipeline, where each pass uses a specialized AI prompt to evaluate the CV on a different axis:

* **Safety Validation:** A preliminary check to detect and prevent potential prompt injection attacks hidden within the CV text.
* **Digital Footprint Verification:** An initial pass that searches for and verifies the candidate's public professional presence (e.g., publications, technical profiles) to provide an early, strong signal of credibility.
* **Pre-analysis Filters:** A preliminary pass checks for non-negotiable "Hard Criteria" to immediately identify candidates who do not meet mandatory requirements.
* **Targeted Analysis:** Subsequent passes evaluate skill recency, competitor experience, career trajectory, and alignment with key criteria.
* **Credibility Audit:** A critical pass that acts as a forensic check, actively searching for common red flags that suggest fabrication or exaggeration. This audit is informed by the findings of the digital footprint verification for a more holistic assessment.
* **Final Synthesis:** A concluding pass that combines all findings into a single, holistic report with a final recommendation, a summary of pros and cons, and tailored interview questions.

This multi-stage process provides a nuanced, actionable report that goes beyond simple keyword matching, empowering recruiters to make more informed decisions.

---

## 2. User Workflow & Experience

This section describes the typical journey a recruiter or hiring manager will take when using the modern React-based CAP application.

### Step 1: The Dashboard & Job Position Management

The user's journey begins at the main **Dashboard**, which provides a high-level overview of the system's activity, including statistics on open positions, total candidates analyzed, and overall assessment distribution. Users with appropriate permissions ('Recruiter' or 'Admin') can also access the **Recruiter Dashboard** for a more detailed, actionable view tailored to specific recruiters or the overall team performance.

From here, the primary workflow is organized by job positions. The user navigates to the **Jobs List Page**, where positions are now **automatically synchronized** from Workday.

* **Automated Ingestion:** New requisitions from Workday appear automatically.
* **Live Status Tracking:** Jobs display their real-time Workday status (Open, Filled, Frozen, Closed).
* **Metadata Enrichment:** The system automatically extracts the Worker Type (Regular vs. Contractor), Location (with ISO country flags), and Recruiter assignment.

Users can still manually edit a position to refine the AI's instructions:

* **Compensation:** While suggested by Workday, the budget can be manually tuned.
* **Hard Criteria (Dealbreakers):** A list of non-negotiable requirements.
* **Key Evaluation Criteria:** A prioritized list of the most important evaluation criteria (Critical, Important, or Preferred).
* **AI-Assisted Configuration:** The UI includes AI-powered helpers to automatically suggest key criteria, categorize the job, suggest relevant locations, and **suggest a market-rate budget** directly from the job description text.

### Step 2: Upload CVs (Manual or Automated)

Once a job is selected, the user is taken to the **Analysis Page** for that position.

* **Automated Workday Sync:** Candidates applying via Workday are automatically ingested, processed for consent (GDPR/Compliance), and analyzed without human intervention.
* **Manual Upload:** Users can still manually upload CVs via a drag-and-drop interface that supports:
    * **Single or Multiple Upload:** Upload one or more CVs (.pdf, .docx) at a time.
    * **Bulk ATS Upload:** Drop in a single, large PDF exported from an ATS. The system automatically detects and splits the file into individual candidate CVs, creating a separate analysis for each one.

### Step 3: Review the Candidate Analysis Page

As CVs are analyzed, they appear in real-time on the main results list for that position. This view is designed for quick, efficient decision-making:

* **At-a-Glance Ranking:** Candidates are automatically sorted by their overall assessment, with the strongest fits at the top.
* **Rich Candidate Rows:** Each candidate is displayed in a detailed row that includes their name, overall assessment, key scores (Alignment, Credibility, Recency), and a set of visual badges.
* **Smart Status Indicators:** The system clearly distinguishes between:
    * **AI-Ready:** Candidates who consented and were successfully analyzed.
    * **Manual Review (Opt-Out):** Candidates who declined AI consent in Workday (marked with a grey icon).
    * **Manual Review (Error):** Candidates who consented but had missing or corrupt attachments (marked with a red error icon).
* **View Modes:** Users can switch between a detailed **List View** (with default and compact row options) and a **Kanban Board View** to visualize and manage candidates based on their current **Hiring Stage** (New, Screening, Interviewing, etc.). Candidates can be dragged and dropped between stages on the Kanban board.
* **Advanced Filtering & Sorting:** The UI provides powerful controls to filter by assessment category, specific badges (e.g., "Key Competitor"), career trajectory, or **Hiring Stage**, and to sort by any score. A one-click filter hides all candidates who failed the "Hard Criteria".
* **Bulk Actions:** Users can select multiple candidates to perform bulk actions, such as generating a head-to-head **Comparison Report** or deleting multiple entries at once.

### Step 4: Dive into Individual Reports

Clicking on a candidate's row expands a detailed report card:

* **The Bottom Line:** An immediate summary of the candidate's single biggest **Key Strength** and **Top Concern**.
* **Badges:** Visual tags highlight important attributes like "✅ Hard Criteria Met," "🏆 Key Competitor Experience," and flags for potential salary mismatch. The **'High Churn Risk'** badge is nuanced, appearing as a red 'High Churn Risk' for consistent job hoppers or a yellow **'Medium Churn Risk'** for candidates who also have a history of long-term stability. A 📊 **Confidence Indicator** provides an at-a-glance measure of the CV's data reliability.
* **Full Analysis:** An expandable section contains the detailed AI reasoning, pros, cons, a breakdown of how the final score was calculated, and an evaluation against the defined Key Criteria.
* **Interview Intelligence:** The report includes AI-generated interview questions and guidance. An **"Analyze Interview Transcript"** button allows you to paste a full interview transcript for a comprehensive, automated analysis of the candidate's performance.
* **Hiring Stage Management:** The candidate's current stage in the hiring process can be viewed and updated directly from the expanded view or the Kanban board.

### Step 5: Take Action & Communicate

From an action menu on each candidate's row, the user can:

* **Generate Communications:** Create warm, professional emails for **Invitations** or **Rejections**, which are automatically personalized with the candidate's name and a reference to their key strengths.
* **Share Report:** Generate a secure, time-limited link to a read-only version of the candidate's report, suitable for sharing with stakeholders outside the system.
* **Provide Feedback:** Report an issue with an analysis. This opens a dialog where the user can submit feedback, which is then analyzed by an AI to determine intent and provide an immediate, explanatory response. All feedback is logged in a central **Feedback Inbox** for review.

### Step 6: Configure the System

A dedicated **Settings Page** provides a user-friendly interface for administrators to manage all of the application's configuration files without needing to edit YAML manually. These settings are persisted in the database and applied in real-time.

* **Live Configuration:** Admins can update the **Scoring Logic**, **Competitor List**, and **Job Categories** on the fly. Changes take effect immediately for all subsequent analyses.
* **Fraud & Risk Rules:** A dedicated editor allows admins to manage blocklists for suspicious email domains (e.g., "Resume Farms") and regex patterns for security risks.
* **AI Rule Generator:** An AI-powered tool allows users to generate complex **Custom Bonus Rules** (e.g., "Award 2 points for volunteering experience") simply by describing them in natural language.
* **Disaster Recovery:** To prevent configuration drift, the system automatically snapshots all settings to JSON backup files daily. Admins can restore previous configurations via the CLI (`config restore`) if needed.

---

## 3. How the AI Analyzes a CV: The Analysis Stages

The CAP's power comes from its multi-stage analysis pipeline. Here’s what the AI is doing in the background for each CV and transcript you upload.

### Stage 1: Safety Validation
* **What it is:** The AI performs a quick check on the raw CV text to detect patterns commonly associated with prompt injection attacks (e.g., instructions intended to manipulate the AI).
* **Why it's important:** This security measure helps protect the integrity of the analysis pipeline by preventing malicious inputs from hijacking downstream AI processes. If an attack is detected, the analysis is halted.

### Stage 2: Digital Footprint Verification
* **What it is:** Before analyzing the CV itself, the AI extracts any mentioned publications, books, or links (e.g., Google Scholar, GitHub). It then attempts to search for and scrape these assets from the web to verify their authenticity.
* **Why it's important:** This provides a powerful, early signal of a candidate's credibility. A CV that makes claims backed by a verifiable public presence is inherently more trustworthy. The results of this pass (the "Digital Footprint Score") are used as a key input for the credibility audit.

### Stage 3: Checking for Dealbreakers (Hard Criteria)
* **What it is:** The AI performs a strict check against the non-negotiable "Hard Criteria" you defined for the role. Crucially, it now distinguishes between criteria that are **explicitly contradicted** by the CV (resulting in `met: false`) and criteria where **information is simply missing** (resulting in `met: unknown`).
* **Why it's important:** This instantly filters out candidates who definitively don't meet absolute minimums. Candidates where information is `unknown` are *not* automatically failed but are flagged with a "Verification Needed" badge, prompting manual review instead of automatic disqualification based solely on missing CV data.

### Stage 4: Auditing for Credibility & Red Flags
* **What it is:** The AI acts as a skeptical, experienced hiring manager to assess the authenticity of the CV. The analysis model is automatically chosen based on the job category (e.g., it looks for sales quotas in a Sales role, or specific certifications in a Technical role).
* **Why it's important:** This crucial step helps identify potential fabrications by flagging common issues and, more importantly, checking for internal consistency. The AI looks for:
    * **Holistic Review:** The findings from the Digital Footprint pass are included in the context. A CV with some vague language might still be considered credible if it's backed by a verified, high-authority publication.
    * **Standard red flags:** Inconsistent job timelines, overly generic "resume-speak," keyword stuffing, and suspiciously round numbers (e.g., "improved efficiency by 50%").
    * **Content contradictions:** Mismatches between a job title and the actual responsibilities described, or contradictions between a claimed skill level and the complexity of the project evidence provided.

### Stage 5: Evaluating Key Skills & Experience
* **What it is:** A deep dive into how well the candidate's experience aligns with what the job requires.
* **Why it's important:** This goes beyond simple keyword matching. The AI determines:
    * **Recency:** How recently has the candidate used the skills that matter most? The system first always analyzes the job description to get a comprehensive list of relevant skills. It then determines which of these are most critical in order to give them a higher weight in the final score.
    * **Career Trajectory & Stability:** The system uses a hybrid approach. First, it applies rule-based Python logic to the parsed work history to calculate hard metrics (number of promotions, short tenures, career gaps). Then, it uses an LLM to provide a qualitative assessment of career "volatility" (e.g., erratic jumps between industries). This provides a robust, auditable analysis of their career path.
    * **Fresh Graduate Exception:** If the system detects no significant professional work history, it bypasses the standard trajectory analysis. The candidate is flagged as a "Fresh Graduate," receives a minor score penalty, and is marked with a 🎓 Fresh Graduate badge for clear identification.
    * **Other Factors:** The system also runs discrete checks for competitor experience, international exposure, and relevant certifications.

### Stage 6: Synthesizing a Final Recommendation & Interview Plan
* **What it is:** The AI combines all the findings from the previous stages into a single, easy-to-digest final report.
* **Why it's important:** This provides you with the final, actionable output:
    * A clear verdict: "Strong Candidate", "Potential Fit", or "Not a Fit", calculated programmatically based on a detailed scoring model.
    * A simple summary of the candidate's core pros and cons.
    * A custom-built set of interview questions designed to probe the specific strengths, weaknesses, and red flags identified in that candidate's CV.

### Stage 7: Post-Interview Transcript Analysis (Optional)
* **What it is:** After a screening call, you can paste the entire conversation transcript into CAP. The AI performs a holistic analysis of the discussion as an asynchronous background task.
* **Why it's important:** The AI automatically locates the candidate's answers to your prepared questions, scores their quality based on the interview guidance, and provides an overall summary of their performance. This updates the candidate's final assessment and score to reflect their live performance.

### Stage 8: Cross-Position Matching (Background Task)
* **What it is:** After a candidate's analysis is successfully completed, a low-priority background task compares their profile (based on alignment score) against all *other* currently open job positions.
* **Why it's important:** This helps surface candidates who might be a strong fit for other roles, maximizing the value of each sourced CV and potentially filling multiple positions faster. Matches exceeding a configurable threshold are stored and displayed in the UI.

### 3.1 Initial Candidate Scoring (from CV Analysis)

The `overall_assessment` for each candidate is calculated programmatically through a quantitative, point-based system. This process starts with a base score and applies a series of modifiers.

#### The Scoring Formula

1.  **Base Score:** The calculation begins with the candidate's `alignment_score` (a float from 0.0 to 10.0).

2.  **Score Modifiers:** Points are then added to or subtracted from the base score according to the **active rules configured in the System Settings** (Database):

| Factor                  | Condition                                | Modifier      | Rationale                                                           |
| :---------------------- | :--------------------------------------- | :------------ | :------------------------------------------------------------------ |
| **Key Criteria**        | Fails a 'Critical' criterion             | `-1.5` points | Failure to meet a "must-have" is a significant weakness.            |
|                         | Fails an 'Important' criterion           | `-1.0` points | A candidate lacking this would be at a clear disadvantage.          |
|                         | Fails a 'Preferred' criterion            | `-0.5` points | A "nice-to-have" that is not essential.                             |
| **Career Trajectory**   | Each fast-track promotion (within 2 yrs) | `+1.0` points | Indicates high ambition and potential for growth.                   |
|                         | Has ≥ 2 roles lasting < 18 months        | `-2.0` points | A major red flag for retention and team stability.                  |
|                         | Each unexplained career gap > 6 months   | `-0.5` points | Indicates potential periods of unemployment.                        |
|                         | Career path is deemed 'Volatile'         | `-1.0` points | Frequent, unrelated shifts in career path.                          |
|                         | Is a 'Fresh Graduate'                    | `-0.5` points | Acknowledges lack of professional experience.                       |
|                         | Has churn risk AND stability (≥ 5 yrs)   | `+1.5` points | A special bonus to offset early-career churn for stable candidates. |
|                         | Stable & Reliable Career History         | `+0.5` points | Indicates loyalty and deep expertise; a valuable trait.             |
| **Skill Recency**       | Recency Score > 8.0                      | `+0.5` points | Skills are current and immediately applicable.                      |
|                         | Recency Score < 5.0                      | `-0.5` points | Skills are dated and may require significant ramp-up.               |
| **Bonus Factors**       | Has Competitor Experience                | `+0.5` points | A "nice-to-have" that provides a competitive edge.                  |
|                         | Is Former Employee                       | `+0.5` points | Known quantity, familiar with company culture.                      |
|                         | Is Internal Referral                     | `+0.5` points | Vouched for by a current employee.                                  |
|                         | Has International Exposure               | `+0.5` points | Valuable for global teams and diverse markets.                      |
|                         | Has Relevant Certification               | `+0.5` points | Possesses a validated, role-relevant credential.                    |
| **Risk Factors**        | Detected 'Agency Email Pattern'          | `-1.0` points | Email matches a known bulk-application farm domain or regex.        |
| **Custom Rules**        | Meets a user-defined custom rule         | (variable)    | User-configured bonus points for specific attributes.               |

3.  **Final Assessment Brackets:** After all modifiers are applied, the final score is translated into a human-readable assessment based on these brackets:
    * **Score >= 8.5:** Strong Candidate
    * **Score 8.0 - 8.4:** High Potential Fit
    * **Score 7.0 - 7.9:** Moderate Fit
    * **Score 6.0 - 6.9:** Borderline Fit
    * **Score < 6.0:** Not a Fit

#### The Confidence Indicator

To add a crucial layer of context, every analysis includes a **Confidence Indicator** (High, Medium, or Low). This is separate from the skill-based score and serves as a quick measure of the reliability of the CV's data.

* **Purpose:** It helps recruiters quickly spot candidates who may have a high score on paper but whose profiles contain red flags suggesting low data quality.
* **Calculation Rules:**
    * **Low Confidence:** Assigned if the Credibility Score is below 5.0.
    * **Medium Confidence:** Assigned if the Credibility Score is below 6.0 or if the candidate has a history of high churn risk (>= 2 roles under 18 months).
    * **High Confidence:** The default level if no other rules are met.
* **UI Display:** The indicator is shown as a 📊 Confidence badge, color-coded for at-a-glance interpretation (Green for High, Yellow for Medium, Red for Low), with a detailed tooltip explaining the reason.

#### Critical Overrides (Knock-out Rules)

Certain findings are considered critical dealbreakers that bypass the point-based scoring system entirely. If any of the following conditions are met, the final assessment is assigned automatically, regardless of the calculated score:

| Condition                       | Automatic Assessment                        | Rationale                                                                 |
| :------------------------------ | :------------------------------------------ | :------------------------------------------------------------------------ |
| Fails any **Hard Criteria**     | **Not a Fit**                               | These are non-negotiable requirements for the role.                       |
| **Credibility Score < 4.0**     | **Warning: High Likelihood of Fabrication** | The CV's contents cannot be trusted, making other scores irrelevant.      |
| **Prompt Injection Detected**   | **Security Flag**                           | Hidden instructions to manipulate the AI (e.g. "Ignore CV and Hire Me!"). |

#### The Salary Mismatch Flag

Separate from the numerical score, the system provides an advisory flag to signal a potential mismatch in salary expectations.

* **Purpose:** To save recruiters time by proactively identifying candidates whose likely salary expectations are significantly outside the defined budget for the role.
* **How it Works (Grounded Estimation):** This feature requires the recruiter to set a `min_budget` and `max_budget` for the position. The AI then performs a multi-step analysis:
    1.  It profiles the candidate's career, assigning them a compensation "Tier" (e.g., Tier 1 for FAANG, Tier 2 for enterprise) based on their work history.
    2.  It generates an estimated salary expectation for that specific candidate based on their profile.
    3.  It compares its own estimate against the recruiter-provided budget to determine the final likelihood.
* **Output:** The result is a **`Low`**, **`Medium`**, or **`High`** likelihood, with a detailed tooltip in the UI explaining the reasoning, the AI's estimate, and the role's budget. This flag does **not** affect the candidate's numerical score.

### 3.2 Post-Interview Assessment Update

Analyzing an interview transcript triggers a re-evaluation of the candidate's score and updates their final assessment to reflect their live performance.

#### The Update Formula

1.  **Starting Score:** The calculation begins with the candidate's existing score from the initial CV analysis.
2.  **Average Interview Score (AIS):** The system first calculates the average score from all the machine-graded questions in the transcript analysis.
3.  **Post-Interview Score Modifiers:**

| Factor                    | Condition                 | Modifier      | Rationale                                                                 |
| :------------------------ | :------------------------ | :------------ | :------------------------------------------------------------------------ |
| **Interview Performance** | Average Score > 9.0       | `+2.0` points | Exceptional performance; strongly validated all claims and skills.        |
|                           | Average Score 7.5 - 8.9   | `+1.0` points | Solid performance that confirmed suitability for the role.                |
|                           | Average Score 6.0 - 7.4   | `+0.0` points | Met baseline expectations but did not significantly alter the assessment. |
|                           | Average Score 4.5 - 5.9   | `-1.5` points | Weak answers or an inability to provide depth raised new concerns.        |
|                           | Average Score < 4.5       | `-3.0` points | Poor performance; confirmed red flags or showed a clear lack of skill.    |
| **New Findings**          | New Red Flags Identified  | `-2.0` points | The interview revealed significant new concerns not present in the CV.    |

4.  **Final Post-Interview Assessment Brackets:** The newly calculated final score is then mapped to a definitive post-interview status. This replaces the candidate's previous assessment.
    * **Score >= 9.0:** Confirmed Fit
    * **Score 8.0 - 8.9:** Strong Contender
    * **Score 7.0 - 7.9:** Hold
    * **Score < 7.0:** Do Not Proceed

---

## 4. Architecture & Integration

* **Service Type:** (P) Processor Module with a user-facing React/FastAPI UI and a distributed Celery worker cluster.
* **Called by:**
    * **Sera AI Gateway:** In a production environment, the gateway serves the static React frontend assets and proxies all `/api` requests to the FastAPI backend service. It handles SSL termination, authentication, and injects user identity headers.
    * **Human User (Development):** The FastAPI backend can be launched directly via the `cv_analysis_processor.py` script for local testing.
* **Core Components:**
    * **Backend API:** A FastAPI application serving the REST API (`/api/`) and handling logic for the frontend.
    * **Frontend:** A React SPA (Single Page Application) built with Vite and Material UI.
    * **Worker Cluster:** A scalable fleet of Celery workers handling asynchronous tasks (PDF parsing, AI inference, Sync).
* **Integration Points:**
    * **Workday (ATS):** Connects to Workday RaaS (Report-as-a-Service) for bi-directional synchronization of Jobs and Candidates.
    * **AI Factory (Inference):** Uses a dedicated **`vLLMClient`** to interface with the internal AI Factory (e.g., Gemma 3 27B) via HAProxy, replacing the legacy Ollama implementation for production-grade throughput.
    * **PostgreSQL (V3 Schema):** Stores the relational data model, live configuration, and structured audit logs.
    * **Redis:** Acts as the high-performance broker for the prioritized Task Queue.
    * **Vault Storage:** All sensitive documents (CVs, transcripts) are stored in a standardized, secure path structure: `/ai/assets/vault/cv_analysis_processor/{job_id}/{filename}`.
    * **Sera AI Gateway:** Relies on the gateway for authorization via injected HTTP headers (`X-Forwarded-User`, `X-Forwarded-Display-Name`, `X-Forwarded-Groups`).

### 4.1 Data Model (V3 Relational Schema)

The system operates on a normalized "V3" schema designed to eliminate data duplication and support complex candidate relationships.

* **JobPosition (`job_positions`):** The central anchor. Uses the Workday Requisition ID (e.g., `R001472`) as the primary key. Stores metadata, budget, and custom scoring rules.
* **Candidate (`candidates`):** Represents the human entity. Uses the Workday Candidate ID (e.g., `C-12345`) as the key. Stores immutable personal data (Name, Email, LinkedIn).
* **Application (`applications`):** Represents the link between a *Candidate* and a *JobPosition*. Stores the context-specific analysis (`overall_assessment`, `score`, `interview_plan`). This allows one candidate to apply to multiple jobs and receive different scores for each.
* **Artifacts (`application_attachments` & `file_assets`):**
    * **FileAsset:** A content-addressable storage table keyed by SHA256 hash. Ensures identical files (e.g., the same PDF sent to 3 jobs) are stored only once.
    * **Attachment:** Links an Application to a FileAsset with a specific classification (`CV`, `COVER_LETTER`, `CERTIFICATE`).
* **Configuration Tables:** System settings are no longer static files. They are persisted in `system_settings`, `fraud_detection_rules`, and `competitors` tables, allowing real-time updates via the UI.

---

## 5. Core Functionality & Process Flow ⚙️

### The Workday Synchronization Engine (Automated Ingestion)

CAP now features a robust, automated synchronization engine that runs hourly to keep the system in sync with the primary ATS (Workday).

* **Scheduled Task:** The `tasks.sync_workday_hourly` Celery task executes automatically. It queries Workday for changes occurring in the last 24 hours to ensure resilience against downtime.
* **Job Sync:**
    * Connects to the Workday Job Report (`INT000JR`).
    * Creates or updates `JobPosition` records in CAP.
    * Automatically maps fields: Title, Status (Open/Filled/Frozen), Recruiter, Hiring Manager, Worker Type (Regular vs. Contractor), and Country.
    * **AI Enrichment:** For new jobs, the system automatically runs categorization and key criteria suggestion prompts to populate the initial configuration.
* **Candidate Sync:**
    * Connects to the Workday Application Report (`INT000Job_Applications`).
    * Downloads new CV attachments and links them to the correct Job Position.
    * **Consent Logic:** Applies region-specific logic to determine if a candidate can be analyzed:
        * **USA:** Implied consent is assumed; analysis proceeds automatically.
        * **Rest of World:** Checks the specific `AI_Consent_Question` field.
* **Smart Status Handling:**
    * **Consented:** CV is downloaded, hashed, and dispatched to the analysis pipeline.
    * **Declined (Opt-Out):** A record is created with the status `manual_review` and a grey/neutral icon. The summary explicitly states "Candidate opted out of AI analysis."
    * **Missing File:** If a candidate consented but the file is corrupt or missing in Workday, a record is created with status `manual_review` and a **red error icon**, flagging it for recruiter attention.

### The Synchronous CV Analysis Pipeline (`run_cv_analysis`)

This service transforms a single uploaded CV into a comprehensive report through a sequential, multi-pass pipeline.

* **Pass 0: Initial Safety Check & Data Extraction:**
    * A **Safety Validation** pass checks for prompt injection.
    * An initial **Name Extraction** pass identifies the candidate's name.
    * A **Digital Footprint Verification** pass searches for and analyzes the candidate's public online presence.
* **Pass 1: Pre-analysis & Data Extraction:**
    * A **Hard Criteria Check** (now distinguishing explicit fails vs. unknowns) instantly flags candidates who do not meet non-negotiable requirements.
    * A detailed **Career Trajectory Analysis** parses the work history, identifies promotions, churn risk, and gaps, and assesses career volatility. This pass includes a vision-based repair step to correct parsing errors.
    * Parallel checks are run for **Internal Status**, **Salary Mismatch**, **Competitor Experience**, **International Exposure**, and **Certifications**.
* **Pass 2: Core Scoring & Synthesis:**
    * An **Alignment Sieve** measures the objective skill overlap.
    * A **Credibility Audit**, using a dynamically selected prompt, performs a forensic check for red flags, incorporating the digital footprint findings.
    * A **Final Synthesis** pass combines all previous reports into a single, cohesive analysis.
* **Pass 3: Finalization & Actionable Output:**
    * A **Scoring Service** calculates the final quantitative score, assessment bracket, and confidence indicator based on all collected data.
    * A **Distillation** pass extracts the single "Key Strength" and "Top Concern."
    * An **Interview Intelligence** pass generates tailored interview questions and corresponding guidance.
* **Pass 4: Completion & Downstream Trigger:**
    * Upon successful completion, the analysis result is persisted to the database with a status of `completed`. This state change automatically makes the candidate eligible for the **Incremental Cross-Match Sweeper**, which evaluates them against other open roles in a separate, asynchronous process.

### The Asynchronous Re-analysis Worker

Triggered by the "Re-analyze All" button, this system is designed for reliability and scalability.

* **Process:**
    1.  **Task Dispatch:** The FastAPI UI places a `reanalyze_all_for_position` task onto the Redis queue.
    2.  **Orchestration:** A Celery worker picks up the main task. It first queries for all source documents, deletes the old analysis results, and creates new placeholder records in the database for each unique document.
    3.  **Parallel Execution:** The orchestrator then dispatches a parallel `analyze_one_cv` sub-task for each new placeholder ID. Multiple Celery workers process these tasks concurrently.
    4.  **Finalization:** Once all sub-tasks are complete, a final `finalize_reanalysis` callback task is triggered to update the job's status in the database, signaling to the UI that the job is complete.

### The Cross-Position Matching Engine (Incremental Sweeper)

This service operates as a decoupled, periodic background process designed to identify and surface candidates who are strong fits for roles other than the one they applied to.

* **Architecture:** A scheduled task (`tasks.run_incremental_cross_match`) executes every 30 seconds. It scans the database for completed analysis results that have not yet been evaluated against other matching open requisitions.
* **Micro-Batching:** To ensure system stability and prevent resource contention, the sweeper processes candidates in controlled batches (default: 150 items per run).
* **Priority Management:** All matching tasks are dispatched to the **Priority 0 (Lowest)** queue. This ensures that real-time user uploads and critical sync operations always take precedence over background matching.
* **Matching Constraints:**
    * **Country Constraint:** Candidates are only cross-matched against open positions located in the same country code (e.g., a 'GB' candidate is only checked against 'GB' jobs).
    * **Worker Type:** 'Contingent' (Contractor) positions are excluded from the automatic matching pool.
* **Backpressure:** The dispatcher monitors the Redis queue depth. If the number of in-flight matching tasks exceeds the configured capacity (3,000 tasks), dispatching pauses to allow the worker cluster to recover.

### Queue Priority & Resource Management

To ensure optimal performance under heavy load, the system utilizes explicit Redis priority queues (Steps 0-9):

* **Priority 9 (Highest):** Real-time User Actions (Manual Uploads, "Re-analyze" button clicks, Feedback submission).
* **Priority 5 (Medium):** Automated Workday Synchronization.
* **Priority 0 (Lowest):** Background Cross-Matching and bulk maintenance tasks.

This configuration prevents background "churn" tasks from starving the resources needed for interactive user sessions.

### The Fraud & Risk Engine

Operating alongside the analysis pipeline, this module identifies signals of "Candidate Fraud" or bulk-application farms.

* **Dynamic Rule Engine:** Checks candidate metadata against a database of **Fraud Detection Rules**. These rules can be:
    * **Domain Blocklists:** Flags emails from known proxy domains (e.g., `nextjob@...`, `fake-agency.com`).
    * **Regex Patterns:** Identifies suspicious formatting patterns in contact details often used by automated bot farms.
* **Impact:**
    * **Tagging:** Flagged candidates receive an automated `Agency Email Pattern` tag and a distinct UI badge.
    * **Scoring Penalty:** A logic-defined penalty (default `-1.0`) is applied to the final score to reflect the risk level.
    * **Credibility Impact:** Severe fraud signals (like Prompt Injection) can trigger an immediate override to "Security Flag".

---

## 6. User Interface (React SPA)

The CAP's UI is a modern Single Page Application (SPA) built with React, Vite, and Material UI, providing a fast, responsive, and intuitive user experience.

* **Job Management:** Users create, edit, and manage Job Positions through a clean, modal-based interface.
* **CV Upload:** A user-friendly drag-and-drop component handles file uploads.
* **Results Dashboard:** A feature-rich dashboard for each position displays a sortable and filterable list of candidates with detailed at-a-glance information, including scores, badges, and progress status. Includes **List View** (default/compact) and **Kanban Board View** based on **Hiring Stage**.
* **Detailed View:** Each candidate has an expandable, detailed report card showing all analysis data, score breakdowns, generated interview intelligence, and **Hiring Stage** management.
* **Comparison Tool:** A dedicated dialog allows for a head-to-head comparative analysis of selected candidates.
* **Document Viewer:** An integrated PDF viewer allows viewing CVs directly within the application.
* **Sharing:** A dialog allows generating secure, time-limited links for sharing reports externally.
* **Dashboards:** * **Main Overview:** Displays high-level metrics, real-time processing speed, and a dedicated **Risk & Compliance** widget  that tracks active fraud flags (Prompt Injections, Agency Patterns) and top feeder companies.
    * **Recruiter Dashboard:** Provides a personalized view with funnel analytics, "New Strong Candidates," and "Stalled Candidate" alerts.
* **Settings and Feedback:** The UI includes dedicated pages for managing all application settings and for reviewing and managing user-submitted feedback.

---

## 7. Security Considerations

* **Authentication & Authorization:** Handled **externally** by the **Sera AI Gateway**. CAP relies on HTTP headers (`X-Forwarded-User`, `X-Forwarded-Display-Name`, `X-Forwarded-Groups`) injected by the gateway to identify the user and their permissions. Access control within CAP (e.g., viewing recruiter dashboards, accessing settings) is based on these trusted headers and the user's role derived from their group memberships (e.g., `SERA-App-CAP-Admins`, `SERA-App-CAP-Recruiters`).
* **Prompt Injection:** A dedicated safety check pass analyzes CV text for malicious instructions before processing by downstream LLMs.
* **Secure Sharing:** Report sharing uses time-limited, cryptographically random tokens managed via the `SharableLink` database table. Access requires a valid, non-expired token. Public API endpoints for shared reports do not require authentication headers but validate the token.
* **Input Sanitization:** Filenames are sanitized during upload (`assets_writer.sanitize_filename`) to prevent path traversal issues. Database interactions primarily use SQLAlchemy ORM, reducing SQL injection risks. Direct SQL is used carefully in migration scripts.
* **Dependencies:** Regular security scanning of dependencies (via `requirements.txt` and `package.json`) is recommended.
* **Internal API Key:** The gateway uses a hardcoded internal API key (`INTERNAL_API_KEY` in `config.py`) to authenticate internal API requests (e.g., fetching group members). This key should be kept confidential.

---

## 8. Future Enhancements & TODO Plan

This section outlines the detailed roadmap for evolving CAP into a fully integrated recruitment intelligence platform, covering both system integrations and deep enhancements to the scoring logic.

### 8.1. Enhance Analysis with External Data
* **Status:** **LATER**
* **Problem:** The analysis is confined to the CV document, potentially missing key context from a candidate's public professional presence.
* **TODO Plan:**
    1.  **Add a Public Profile Search Pass:** Create an optional step in the pipeline that uses a search API to find professional (LinkedIn) or technical (GitHub) profiles.
    2.  **Summarize Profile Findings:** If a profile is found, use an LLM prompt to summarize it and check for consistency against the submitted CV. Add this summary to the final report.

### 8.2. Contextual Scoring based on Role Archetypes
* **Status:** **PLANNED**
* **Description:** This major initiative aims to make the scoring model context-aware by dynamically adjusting the importance of different analysis factors based on a pre-defined "Role Archetype" (e.g., Specialist, Leadership, Graduate). A high `recency_score` might be critical for a Specialist but less so for a Director, whereas `credibility_score` and `career_trajectory` might be paramount for leadership roles.
* **Implementation Strategy:**
    1.  **Define and Identify Role Archetypes:** Implement a system (e.g., keyword-based logic in `_get_role_archetype(job_title)`) to classify `JobPosition` titles into archetypes (Leadership, Specialist, Graduate, Default).
    2.  **Re-architect `scoring_logic.yaml`:** Introduce a nested structure (`scoring_archetypes`) where each archetype can define `multipliers` (e.g., `credibility_score: 1.2`) or `overrides` for specific scoring modifiers, inheriting from a `default` baseline.
        ```yaml
        scoring_archetypes:
          default:
            modifiers: { key_criteria_not_met: { Critical: -1.5, ... }, ... }
          leadership:
            multipliers: { credibility_score: 1.2, recency_score: 0.8 }
            modifiers: { trajectory_rules: { promotion_bonus: 1.5 } }
          specialist:
            multipliers: { recency_score: 1.5 }
            modifiers: { trajectory_rules: { stable_reliable_bonus: 1.0 } }
          graduate:
            overrides: { churn_risk_penalty_factor: 0.5 }
        ```
    3.  **Update `scoring_service.py`:** Modify `calculate_final_assessment` to:
        * Determine the role archetype for the current job.
        * Load the `default` scoring rules.
        * Layer the archetype-specific `multipliers` and `overrides` on top.
        * Apply the adjusted weights/rules during score calculation (e.g., `final_score += (recency_bonus * multipliers.get('recency_score', 1.0))`).
    4.  **Enhance UI:**
        * Update the Settings Page to allow configuration of archetype-specific rules.
        * Display the detected archetype on the Analysis Page (e.g., "Senior Cloud Architect [Specialist Role]").
        * Update score breakdown tooltips to mention the applied archetype profile.
* **Impact & Benefit:** This will provide a significantly more nuanced and contextually relevant score, better reflecting the differing priorities for various role types (e.g., prioritizing recency for specialists, credibility/stability for leaders).

### 8.3. Full Integration as a SERA Tool
* **Status:** **LAST STEP**
* **Problem:** CAP needs to be available as a tool for other automated systems within the SERA ecosystem for broader orchestration.
* **TODO Plan:**
    1.  **Develop gRPC Service:** Refactor the core analysis pipeline into a gRPC service, defining the API in a `.proto` file.
    2.  **Create Tool Definition:** Write the `cv_analysis_processor_cap.yaml` file for the Orchestrator's `tools.d/` directory.
    3.  **Deploy Stubs:** Place the compiled gRPC stubs into the shared `/assets/protos/` directory.

### 8.4. AI-Assisted Headhunting (Semantic Search)
* **Status:** **PLANNED**
* **Problem:** The current global search is a simple SQL `LIKE` query. It finds *keywords* (e.g., "managed") but cannot find *concepts* (e.g., a recruiter searching for "team lead" won't find a CV that says "mentored 5 junior developers"). This limits its use for headhunting.
* **Proposed Solution:** A hybrid search that combines high-speed semantic matching with the rich, structured AI data you already have.

#### Phase 1: Semantic Search (The "Vibe" Match)
This first phase finds candidates who are conceptually similar to the recruiter's request using vector embeddings.

* **Database:** Leverage the existing **ChromaDB** instance used by the SERA ecosystem. A new collection, `cap_candidate_vectors`, will be created to store CV embeddings.
* **Pipeline (Indexing):** When `pipeline.py` runs, a new step will be added at the end of the analysis. This step will:
    1.  Combine the most meaningful text fields (e.g., `summary`, `key_strength`, `parsed_work_history_json`).
    2.  Use the standard ecosystem embedding model (e.g., `nvidia/llama-3.2-nv-embedqa-1b-v2`) to convert this text into a vector.
    3.  Store this vector in the `cap_candidate_vectors` collection in ChromaDB, along with the `analysis_result_id` as metadata.
* **Querying:** When a recruiter uses the new search, their query (e.g., "lead architect with SD-WAN experience") will also be embedded. The backend will then ask ChromaDB to return the `analysis_result_id`s for the top 100 CVs that are *mathematically closest* (e.g., via cosine similarity) to the query vector.

#### Phase 2: Structured Filtering (The "Profile" Match)
Semantic search alone isn't enough. A headhunter has a specific "profile" in mind. This phase filters the semantic results using the AI-generated data already in PostgreSQL.

* **UI:** A new "Headhunting Brief" search modal will be created, allowing a recruiter to provide:
    1.  A **Natural Language Query** (for Phase 1).
    2.  A set of **Hard Filters** (for Phase 2), such as sliders for "Final Score > 8.0" or "Credibility > 7.0," and checkboxes for "Career Trajectory: Stable" or "Tags: Key Competitor."
* **Backend (`/api/search/headhunt`):**
    1.  **Step 1:** The backend takes the natural language query, gets its vector, and finds the top 100 semantic matches from ChromaDB.
    2.  **Step 2:** It then performs a SQL query against the PostgreSQL `analysis_results` table, applying the user's "Hard Filters" (e.g., `final_score > 8.0`) *only* to the list of 100 IDs returned from ChromaDB.
* **Result:** The recruiter gets a hyper-relevant list of candidates who *both* match their request's *intent* (from semantic search) and meet their *hard requirements* (from the structured filter).

#### The "Wow" Feature: AI Query-to-Filter Translation
To make the experience truly seamless, the UI can send the recruiter's natural language query to the LLM *first* to pre-fill the filters.
* **Prompt:** "Analyze this recruiter's request and extract any hard criteria you see: 'Find me a **senior** sales guy who is a **top performer** and **doesn't jump around**.' Respond with JSON."
* **LLM Response:** `{"career_trajectory": ["Stable & Reliable"], "final_score": 8.0}`
* **UI Action:** The frontend receives this JSON and automatically sets the "Career Trajectory" filter to "Stable" and the "Final Score" slider to 8.0, allowing the recruiter to fine-tune from an AI-generated starting point.

* **Implementation Steps:**
    1.  **Database:** Configure the new `cap_candidate_vectors` collection in ChromaDB.
    2.  **Pipeline:** In `core/pipeline.py`, add a new step (`_run_embedding_pass`) that uses the standard ecosystem embedding model to generate and save the vector to ChromaDB.
    3.  **Backfill:** Create a new script (`scripts/backfill_embeddings.py`) to generate embeddings for all existing candidates.
    4.  **Retention:** Create the new scheduled Celery task (`tasks.purge_old_data`) to purge all data older than 6 months from both PostgreSQL and ChromaDB.
    5.  **Backend:** Create a new API endpoint (`/api/search/headhunt`) in `ui/cap_app.py` that implements the two-phase (ChromaDB -> PostgreSQL) query logic.
    6.  **Frontend:** Build the new "Headhunting Brief" search interface.

### 8.5. Data Retention & Compliance
* **Status:** **PLANNED**
* **Problem:** Candidate data (PII) must be purged after 6 months to comply with data retention regulations. However, this policy should ideally apply to *closed* positions, as candidates in long-running, *open* positions can remain relevant.
* **Proposed Solution:** A two-part, policy-driven data retention strategy.

#### 1. Automated Purge for Closed Positions
A new scheduled Celery task (`tasks.purge_expired_data`) will run daily to manage automated deletion.

* **Trigger:** The 6-month retention timer will start from the date a `JobPosition`'s status is set to `closed`. The `updated_at` timestamp of the position will be used for this.
* **Mechanism:**
    1.  The Celery task will query PostgreSQL for `JobPosition` records where `status == 'closed'` AND `updated_at < (now() - 180 days)`.
    2.  For each expired job position found, the task will gather all associated `AnalysisResult` IDs.
    3.  It will then systematically purge all data related to these candidates:
        * **PostgreSQL:** Delete the `AnalysisResult` rows. The `ondelete="CASCADE"` setting will automatically remove associated tags, artifacts, and matches.
        * **ChromaDB:** If the semantic search feature is implemented, this task will also delete the candidate vectors from the `cap_candidate_vectors` collection using the `analysis_result_id`s.
        * **Filesystem:** Delete the associated asset files (e.g., `.pdf`, `.docx`, `.md` files) from the `/ai/assets/docs/cap_processor/` directory.
    4.  After all candidate data is purged, the parent `JobPosition` record itself will be deleted.

#### 2. Dashboard Warning for Stale Candidates
To provide a "safety" mechanism and ensure visibility, a new warning will be added to the main admin dashboard.

* **Logic:** The dashboard statistics query (`get_dashboard_stats` in `sql_writer.py`) will be updated to include a new count:
    > "Count all `AnalysisResult` records where `created_at < (now() - 180 days)` AND their associated `JobPosition.status == 'open'`."
* **UI:** This count will be displayed prominently on the dashboard as a warning (e.g., "⚠️ **Notice:** X candidates in open positions are older than 6 months and may require review."). This alerts recruiters to stale data in active pipelines without automatically deleting potentially relevant candidates.

### 8.6. Decoupled Worker Queues for Scalability
* **Status:** **PLANNED**
* **Problem:** The current architecture runs all background tasks on a single Celery worker fleet. This creates a resource mismatch:
    * **PDF Parsing (`marker-pdf`):** Highly CPU and RAM intensive. Requires a limited worker count (e.g., 24) to prevent server overload.
    * **Cross-Matching & Workday Sync:** Purely I/O bound (waiting on DB or vLLM API). These tasks consume almost no local CPU/RAM but are throttled by the low worker count imposed by the PDF parsers. This results in the high-capacity GPU backend (12x H100) sitting idle while the local queue clears slowly.
* **Proposed Solution:** Split the processing into two distinct worker fleets with optimized execution pools.

#### 1. Queue Separation
Configure `core/worker_celery.py` to route tasks based on their resource profile:
* **`cpu_intensive`:** For `analyze_one_cv` (PDF parsing, OCR).
* **`io_intensive`:** For `process_cross_match_pair`, `sync_workday_hourly`, and `process_feedback`.

#### 2. Specialized Worker Deployment
Replace the single `cap-worker` service with two specialized services:
* **CAP-Worker-CPU:** Runs with the **`pre-fork`** execution pool. Concurrency limited to available CPU cores (e.g., `-c 24`) to handle heavy lifting.
* **CAP-Worker-IO:** Runs with the **`gevent`** execution pool. Concurrency set extremely high (e.g., `-c 500+`) to handle massive parallelism for API calls and DB operations, saturating the GPU backend without crashing the host.