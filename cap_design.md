# CV Analysis Processor (CAP) - Design Document

**Version:** 2.0
**Date:** 2025-10-05
**Author(s):** Martin Kecka, Gemini AI
**Category:** Processor (P)
**Acronym:** CAP

---

## 1. Introduction & Purpose

The **CV Analysis Processor (CAP)** is a specialized AI-driven module designed to automate the initial screening of candidate resumes. Its primary function is to provide hiring managers with a rapid, consistent, and in-depth analysis of a candidate's suitability for a given role by comparing their CV against the job description.

The CAP operates on a multi-pass analysis pipeline, where each pass uses a specialized AI prompt to evaluate the CV on a different axis:

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

The user's journey begins at the main **Dashboard**, which provides a high-level overview of the system's activity, including statistics on open positions, total candidates analyzed, and overall assessment distribution.

From here, the primary workflow is organized by job positions. The user navigates to the **Jobs List Page**, where they can either create a new position or select an existing one from a responsive, card-based interface with filtering and sorting capabilities.

When creating or editing a position, a modal dialog appears, allowing the user to provide essential details:

* **Job Title** and **Job Description**.
* **Compensation:** An optional budget range for the role (`min_budget`, `max_budget`, `currency`).
* **Hard Criteria (Dealbreakers):** A list of non-negotiable requirements.
* **Key Evaluation Criteria:** A prioritized list of the most important evaluation criteria (Critical, Important, or Preferred).
* **AI-Assisted Configuration:** The UI includes AI-powered helpers to automatically suggest key criteria, categorize the job, and suggest relevant locations and a market-rate budget directly from the job description text.

### Step 2: Upload CVs

Once a job is selected, the user is taken to the **Analysis Page** for that position. CVs can be uploaded via a drag-and-drop interface that supports:

* **Single or Multiple Upload:** Upload one or more CVs (.pdf, .docx) at a time.
* **Bulk ATS Upload:** Drop in a single, large PDF exported from an ATS like Workday. The system automatically detects and splits the file into individual candidate CVs, creating a separate analysis for each one.

### Step 3: Review the Candidate Analysis Page

As CVs are analyzed, they appear in real-time on the main results list for that position. This view is designed for quick, efficient decision-making:

* **At-a-Glance Ranking:** Candidates are automatically sorted by their overall assessment, with the strongest fits at the top.
* **Rich Candidate Rows:** Each candidate is displayed in a detailed row that includes their name, overall assessment, key scores (Alignment, Credibility, Recency), and a set of visual badges.
* **Advanced Filtering & Sorting:** The UI provides powerful controls to filter by assessment category, specific badges (e.g., "Key Competitor"), or career trajectory, and to sort by any score. A one-click filter hides all candidates who failed the "Hard Criteria".
* **Bulk Actions:** Users can select multiple candidates to perform bulk actions, such as generating a head-to-head **Comparison Report** or deleting multiple entries at once.
* **Asynchronous Feedback:** The UI provides immediate feedback and progress indicators for long-running background jobs like a full "Re-analyze All" task, allowing the user to continue their work or even close the browser while processing completes reliably on the server.

### Step 4: Dive into Individual Reports

Clicking on a candidate's row expands a detailed report card:

* **The Bottom Line:** An immediate summary of the candidate's single biggest **Key Strength** and **Top Concern**.
* **Badges:** Visual tags highlight important attributes like "✅ Hard Criteria Met," "🏆 Key Competitor Experience," and flags for potential salary mismatch or high churn risk. A 📊 **Confidence Indicator** provides an at-a-glance measure of the CV's data reliability.
* **Full Analysis:** An expandable section contains the detailed AI reasoning, pros, cons, a breakdown of how the final score was calculated, and an evaluation against the defined Key Criteria.
* **Interview Intelligence:** The report includes AI-generated interview questions and guidance. An **"Analyze Interview Transcript"** button allows you to paste a full interview transcript for a comprehensive, automated analysis of the candidate's performance.

### Step 5: Take Action & Communicate

From an action menu on each candidate's row, the user can:

* **Generate Communications:** Create warm, professional emails for **Invitations** or **Rejections**, which are automatically personalized with the candidate's name and a reference to their key strengths.
* **Provide Feedback:** Report an issue with an analysis. This opens a dialog where the user can submit feedback, which is then analyzed by an AI to determine intent and provide an immediate, explanatory response. All feedback is logged in a central **Feedback Inbox** for review.

### Step 6: Configure the System

A dedicated **Settings Page** provides a user-friendly interface for administrators to manage all of the application's configuration files without needing to edit YAML manually. This includes managing the competitor list, job categories, and the scoring logic itself, including an AI-powered tool to generate new **Custom Bonus Rules** from natural language descriptions.

---

## 3. How the AI Analyzes a CV: The Analysis Stages

The CAP's power comes from its multi-stage analysis pipeline. Here’s what the AI is doing in the background for each CV and transcript you upload.

### Stage 1: Digital Footprint Verification

* **What it is:** Before analyzing the CV itself, the AI extracts any mentioned publications, books, or links (e.g., Google Scholar, GitHub). It then attempts to search for and scrape these assets from the web to verify their authenticity.
* **Why it's important:** This provides a powerful, early signal of a candidate's credibility. A CV that makes claims backed by a verifiable public presence is inherently more trustworthy. The results of this pass (the "Digital Footprint Score") are used as a key input for the credibility audit.

### Stage 2: Checking for Dealbreakers (Hard Criteria)

* **What it is:** The AI performs a strict, literal check against the non-negotiable "Hard Criteria" you defined for the role.
* **Why it's important:** This instantly filters out candidates who don't meet absolute minimums (like a specific license or language fluency). It saves you from wasting time on clearly unqualified applicants and is the basis for the most powerful filter on the results page.

### Stage 3: Auditing for Credibility & Red Flags

* **What it is:** The AI acts as a skeptical, experienced hiring manager to assess the authenticity of the CV. The analysis model is automatically chosen based on the job category (e.g., it looks for sales quotas in a Sales role, or specific certifications in a Technical role).
* **Why it's important:** This crucial step helps identify potential fabrications by flagging common issues and, more importantly, checking for internal consistency. The AI looks for:
    * **Holistic Review:** The findings from the Digital Footprint pass are included in the context. A CV with some vague language might still be considered credible if it's backed by a verified, high-authority publication.
    * **Standard red flags:** Inconsistent job timelines, overly generic "resume-speak," keyword stuffing, and suspiciously round numbers (e.g., "improved efficiency by 50%").
    * **Content contradictions:** Mismatches between a job title and the actual responsibilities described, or contradictions between a claimed skill level and the complexity of the project evidence provided.

### Stage 4: Evaluating Key Skills & Experience

* **What it is:** A deep dive into how well the candidate's experience aligns with what the job requires.
* **Why it's important:** This goes beyond simple keyword matching. The AI determines:
    * **Recency:** How recently has the candidate used the skills that matter most? The system first always analyzes the job description to get a comprehensive list of relevant skills. It then determines which of these are most critical in order to give them a higher weight in the final score.
    * **Career Trajectory & Stability:** The system uses a hybrid approach. First, it applies rule-based Python logic to the parsed work history to calculate hard metrics (number of promotions, short tenures, career gaps). Then, it uses an LLM to provide a qualitative assessment of career "volatility" (e.g., erratic jumps between industries). This provides a robust, auditable analysis of their career path.
    * **Fresh Graduate Exception:** If the system detects no significant professional work history, it bypasses the standard trajectory analysis. The candidate is flagged as a "Fresh Graduate," receives a minor score penalty, and is marked with a 🎓 Fresh Graduate badge for clear identification.
    * **Other Factors:** The system also runs discrete checks for competitor experience, international exposure, and relevant certifications.

### Stage 5: Synthesizing a Final Recommendation & Interview Plan

* **What it is:** The AI combines all the findings from the previous stages into a single, easy-to-digest final report.
* **Why it's important:** This provides you with the final, actionable output:
    * A clear verdict: "Strong Candidate", "Potential Fit", or "Not a Fit", calculated programmatically based on a detailed scoring model.
    * A simple summary of the candidate's core pros and cons.
    * A custom-built set of interview questions designed to probe the specific strengths, weaknesses, and red flags identified in that candidate's CV.

### Stage 6: Post-Interview Transcript Analysis (Optional)

* **What it is:** After a screening call, you can paste the entire conversation transcript into CAP. The AI performs a holistic analysis of the discussion as an asynchronous background task.
* **Why it's important:** The AI automatically locates the candidate's answers to your prepared questions, scores their quality based on the interview guidance, and provides an overall summary of their performance. This updates the candidate's final assessment and score to reflect their live performance.

### 3.1 Initial Candidate Scoring (from CV Analysis)

The `overall_assessment` for each candidate is calculated programmatically through a quantitative, point-based system. This process starts with a base score and applies a series of modifiers.

#### The Scoring Formula

1.  **Base Score:** The calculation begins with the candidate's `alignment_score` (a float from 0.0 to 10.0).

2.  **Score Modifiers:** Points are then added to or subtracted from the base score according to the rules in `scoring_logic.yaml`:

| Factor                  | Condition                                | Modifier      | Rationale                                                          |
| :---------------------- | :--------------------------------------- | :------------ | :----------------------------------------------------------------- |
| **Key Criteria**        | Fails a 'Critical' criterion             | `-1.5` points | Failure to meet a "must-have" is a significant weakness.           |
|                         | Fails an 'Important' criterion           | `-1.0` points | A candidate lacking this would be at a clear disadvantage.         |
|                         | Fails a 'Preferred' criterion            | `-0.5` points | A "nice-to-have" that is not essential.                            |
| **Career Trajectory**   | Each fast-track promotion (within 2 yrs) | `+1.0` points | Indicates high ambition and potential for growth.                  |
|                         | Has ≥ 2 roles lasting < 18 months        | `-2.0` points | A major red flag for retention and team stability.                 |
|                         | Each unexplained career gap > 6 months   | `-0.5` points | Indicates potential periods of unemployment.                       |
|                         | Career path is deemed 'Volatile'         | `-1.0` points | Frequent, unrelated shifts in career path.                         |
|                         | Is a 'Fresh Graduate'                    | `-0.5` points | Acknowledges lack of professional experience.                      |
|                         | Stable & Reliable Career History         | `+0.5` points | Indicates loyalty and deep expertise; a valuable trait.            |
| **Skill Recency**       | Recency Score > 8.0                      | `+0.5` points | Skills are current and immediately applicable.                     |
|                         | Recency Score < 5.0                      | `-0.5` points | Skills are dated and may require significant ramp-up.              |
| **Bonus Factors**       | Has Competitor Experience                | `+0.5` points | A "nice-to-have" that provides a competitive edge.                 |
|                         | Is Former Employee                       | `+0.5` points | Known quantity, familiar with company culture.                     |
|                         | Is Internal Referral                     | `+0.5` points | Vouched for by a current employee.                                 |
|                         | Has International Exposure               | `+0.5` points | Valuable for global teams and diverse markets.                     |
|                         | Has Relevant Certification               | `+0.5` points | Possesses a validated, role-relevant credential.                   |
| **Custom Rules**        | Meets a user-defined custom rule         | (variable)    | User-configured bonus points for specific attributes.              |

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

Certain findings are considered critical dealbreakers that bypass the point-based scoring system entirely.

| Condition                       | Automatic Assessment                        | Rationale                                                            |
| :------------------------------ | :------------------------------------------ | :------------------------------------------------------------------- |
| Fails any **Hard Criteria**     | **Not a Fit**                               | These are non-negotiable requirements for the role.                  |
| **Credibility Score < 4.0**     | **Warning: High Likelihood of Fabrication** | The CV's contents cannot be trusted, making other scores irrelevant. |

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

* **Service Type:** (P) Processor Module with a user-facing React/FastAPI UI and a background worker service.
* **Called by:**
    * **Sera AI Gateway:** In a production environment, the gateway serves the static React frontend assets and proxies all `/api` requests to the FastAPI backend service.
    * **Human User (Development):** The FastAPI backend can be launched directly via the `cv_analysis_processor.py` script for local testing.
* **Interacts with:**
    * **LLM Service (AI Factory):** The CAP relies on the configured LLM model for its analysis and generation capabilities.
    * **Shared PostgreSQL Database:** For persistent storage of job positions, candidate data, and analysis results.
    * **Shared Redis Instance:** Acts as the message broker and result backend for the Celery task queue.
    * **CAP Celery Worker:** A dedicated, persistent background process that consumes tasks from the Redis queue to perform heavy processing without blocking the user interface.
    * **Shared Assets Storage:** For storing uploaded CVs, extracted text, and other artifacts.

---

## 5. Core Functionality & Process Flow ⚙️

### The Synchronous CV Analysis Pipeline (`run_cv_analysis`)

This service transforms a single uploaded CV into a comprehensive report through a sequential, multi-pass pipeline.

* **Pass 0: Initial Data Extraction & Verification:**
    * An initial **Name Extraction** pass identifies the candidate's name.
    * A **Digital Footprint Verification** pass searches for and analyzes the candidate's public online presence.
* **Pass 1: Pre-analysis & Data Extraction:**
    * A **Hard Criteria Check** instantly flags candidates who do not meet non-negotiable requirements.
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

### The Asynchronous Re-analysis Worker

Triggered by the "Re-analyze All" button, this system is designed for reliability and scalability.

* **Process:**
    1.  **Task Dispatch:** The FastAPI UI places a `reanalyze_all_for_position` task onto the Redis queue.
    2.  **Orchestration:** A Celery worker picks up the main task. It first queries for all source documents, deletes the old analysis results, and creates new placeholder records in the database for each unique document.
    3.  **Parallel Execution:** The orchestrator then dispatches a parallel `analyze_one_cv` sub-task for each new placeholder ID. Multiple Celery workers process these tasks concurrently.
    4.  **Finalization:** Once all sub-tasks are complete, a final `finalize_reanalysis` callback task is triggered to update the job's status in the database, signaling to the UI that the job is complete.

---

## 6. User Interface (React SPA)

The CAP's UI is a modern Single Page Application (SPA) built with React, Vite, and Material UI, providing a fast, responsive, and intuitive user experience.

* **Job Management:** Users create, edit, and manage Job Positions through a clean, modal-based interface.
* **CV Upload:** A user-friendly drag-and-drop component handles file uploads.
* **Results Dashboard:** A feature-rich dashboard for each position displays a sortable and filterable list of candidates with detailed at-a-glance information, including scores, badges, and progress status.
* **Detailed View:** Each candidate has an expandable, detailed report card showing all analysis data, score breakdowns, and generated interview intelligence.
* **Comparison Tool:** A dedicated dialog allows for a head-to-head comparative analysis of selected candidates.
* **Settings and Feedback:** The UI includes dedicated pages for managing all application settings and for reviewing and managing user-submitted feedback.


---

## 7. Future Enhancements & TODO Plan

This section outlines the detailed roadmap for evolving CAP into a fully integrated recruitment intelligence platform, covering both system integrations and deep enhancements to the scoring logic.

### 7.1. Cross-Position Candidate Matching
* **Status:** **PENDING WORKDAY**
* **Problem:** A great candidate might be a poor fit for one role but a perfect fit for another open position. The system currently has no way to identify these opportunities automatically.
* **TODO Plan:**
    1.  **Create a Matching Service:** Develop a new background service that is triggered after a candidate analysis is successfully completed.
    2.  **Implement Similarity Search:** The service will take the candidate's skills and experience vector and compare it against all other open `JobPosition` descriptions in the database using a similarity metric (e.g., cosine similarity).
    3.  **Flag Potential Matches:** If a candidate's alignment score for another position exceeds a certain threshold (e.g., > 7.0), flag this in the database.
    4.  **Enhance UI:** In the candidate results view, add a notification like "This candidate may also be a strong fit for 'Senior Python Developer (REQ-457)'".

### 7.2. Enhance Analysis with External Data
* **Status:** **LATER**
* **Problem:** The analysis is confined to the CV document, potentially missing key context from a candidate's public professional presence.
* **TODO Plan:**
    1.  **Add a Public Profile Search Pass:** Create an optional step in the pipeline that uses a search API to find professional (LinkedIn) or technical (GitHub) profiles.
    2.  **Summarize Profile Findings:** If a profile is found, use an LLM prompt to summarize it and check for consistency against the submitted CV. Add this summary to the final report.

### 7.3. ATS (Workday) Integration
* **Status:** **PENDING ACCESS**
* **Problem:** To fully streamline the workflow, CAP must eventually integrate with the primary system recruiters use daily, eliminating manual data entry.
* **TODO Plan:**
    1.  **Develop an ATS Connector Service:** Create a dedicated service to handle API communication with Workday (or other ATS).
    2.  **Build a Job Sync Feature:** Add functionality to import `JobPosition` data directly from the ATS instead of creating it manually.
    3.  **Implement Analysis "Push-Back":** Add a "Push to Workday" button that sends the final scores and summary as a note to the candidate's profile in the ATS.

### 7.4. Inject Safety Validation
* **Status:** **PLANNED**
* **Problem:** CVs can contain injected "prompts" designed to confuse or manipulate the LLM, and a proper mechanism for detection is needed.
* **TODO Plan:**
    1.  **Develop Strategy:** Create a new, simple pre-analysis pass to validate the CV text for prompt injection attacks. This can be integrated with the "Hard Criteria" check for efficiency.

### 7.5. Full Integration as a SERA Tool
* **Status:** **LAST STEP**
* **Problem:** CAP needs to be available as a tool for other automated systems within the SERA ecosystem for broader orchestration.
* **TODO Plan:**
    1.  **Develop gRPC Service:** Refactor the core analysis pipeline into a gRPC service, defining the API in a `.proto` file.
    2.  **Create Tool Definition:** Write the `cv_analysis_processor_cap.yaml` file for the Orchestrator's `tools.d/` directory.
    3.  **Deploy Stubs:** Place the compiled gRPC stubs into the shared `/assets/protos/` directory.

### 7.6. Refine and Expand Scoring Model
* **Status:** **WORK IN PROGRESS**
* **Description:** This is a major ongoing initiative to continuously improve the nuance, accuracy, and transparency of the quantitative scoring model. Below are the key sub-initiatives.

#### 7.6.1. Prioritized Key Criteria Scoring
* **Status:** **DONE**
* **Suggestion:** Instead of applying a single, flat penalty for any missed "Key Criterion," the system now uses a prioritized, weighted penalty system. This allows hiring managers to rank up to three criteria, with penalties adjusted accordingly (e.g., -1.5 for #1, -1.0 for #2, -0.5 for #3).
* **Impact & Benefit:** This required UI changes for ranking and backend formula modifications. It makes the final score a much more accurate reflection of the role's specific needs, as defined by the hiring manager.

#### 7.6.2. Refined Credibility Score with Verification Bonus
* **Status:** **3/4 DONE**
* **Suggestion:** The credibility score now focuses on content consistency and verifiable data. A mismatched or limited LinkedIn profile no longer incurs a penalty. Instead, a validated LinkedIn profile that confirms the CV's claims now grants a **+0.5 point bonus**.
* **Impact & Benefit:** This necessitated an optional "LinkedIn Profile URL" field and a new analysis pass. It rewards candidates with a verifiable public presence without unfairly punishing those who don't maintain a detailed profile.

#### 7.6.3. Rule-Based Career Trajectory Analysis
* **Status:** **DONE**
* **Suggestion:** The subjective "Career Trajectory" assessment has been deconstructed into measurable subfactors, including a cumulative **+1.0 point bonus** for each validated promotion within a 2-year timeframe and a "High Churn Risk" calculation based on roles under 18 months.
* **Impact & Benefit:** This required a more sophisticated analysis pass for identifying promotions and parsing dates. The trajectory score is now less subjective and more directly tied to auditable events.

#### 7.6.4. Expanded Bonus Factors
* **Status:** **HALF-DONE**
* **Suggestion:** The list of bonus factors has been expanded to include Referrals, International Exposure, and scores for recent certifications or education quality.
* **Impact & Benefit:** Implementation complexity varies by factor. This allows the system to reward a wider range of desirable attributes beyond the core job description.

#### 7.6.5. Salary Expectations Mismatch Flag
* **Status:** **DONE**
* **Suggestion:** A new non-scoring advisory flag warns recruiters of a potential salary mismatch. The AI estimates the candidate's likely expectation based on their career tier and compares it against the user-defined budget, providing a "Low," "Medium," or "High" mismatch likelihood. This saves significant time by helping recruiters prioritize candidates likely to be within the compensation band.
* **Further Imporvements:** Considering using info from glassdoor, levels.fyi to enchance salary guessing.

#### 7.6.6. Contextualized Assessment Brackets & Confidence
* **Status:** **PENDING ROLE THRESHOLDS**
* **Suggestion:** This major structural change includes three parts: 1) expanding assessment labels to a 5-tier system (Strong, High Potential, etc.), 2) adding a separate "Confidence Indicator" (Low/Medium/High) based on data reliability, and 3) allowing Role-Specific Thresholds, so a junior role can have different scoring brackets than a leadership role.
* **Impact & Benefit:** This is a best-practice approach that provides recruiters with a much richer, more contextualized final assessment, allowing them to make faster and more confident decisions. The combination of a score, a label, and a confidence level is a significant step forward.

### 7.7.1 Shareable Read-Only Report Links
* **Issue:** A hiring manager or stakeholder who isn't a primary user of the tool might need to see a candidate's report.
* **How it would work:** Add a "Share" button to each candidate's report. This would generate a unique, time-limited URL. Anyone with the link could view a read-only, web-based version of that specific candidate's analysis report without needing to log in.
* **Why it's useful:** This dramatically improves collaboration by making it easy to share detailed, AI-powered insights with the wider hiring team.

Draft to 7.6.6 ->
1. The Core Concept: Contextual Scoring Weights
The goal is to dynamically adjust the importance of different analysis factors based on a pre-defined "Role Archetype" (Specialist, Leadership, etc.). A high recency_score is critical for a Specialist but less so for a Director, whereas a high credibility_score and a strong career_trajectory are paramount for a Director.

This means we will be applying multipliers or overrides to the scoring modifiers defined in scoring_logic.yaml.

2. Implementation Strategy: A 3-Step Plan
This is a significant backend change that touches your configuration, scoring logic, and how you identify roles.

Step 1: Define and Identify Role Archetypes
First, we need a system to classify a JobPosition into one of the archetypes. We can do this with a simple, configurable keyword-based approach.

Logic: A new function, let's call it _get_role_archetype(job_title), would scan the job title for keywords in a specific order of precedence.

Keyword Mapping (Example):

Leadership: manager, director, head of, lead, vp

Specialist: specialist, principal, architect, expert, senior

Graduate: graduate, junior, intern, associate

Default: If no keywords match, it falls back to a default archetype.

Step 2: Re-architect the scoring_logic.yaml Configuration
This is the most critical change. The current flat structure of modifiers needs to be nested to support different profiles. We would introduce a new top-level key, scoring_archetypes.

New scoring_logic.yaml Structure (Conceptual):

YAML

scoring_archetypes:
  default:
    # This block contains the current modifier values as the baseline
    modifiers:
      key_criteria_not_met: { Critical: -1.5, ... }
      trajectory_rules: { promotion_bonus: 1.0, churn_risk_penalty: -2.0, ... }
      recency: { high_bonus: 0.5, low_penalty: -0.5 }
      # ... and so on

  leadership:
    # This block contains ONLY the values that are different from the default
    multipliers:
      credibility_score: 1.2  # Makes credibility 20% more impactful
      recency_score: 0.8      # Makes recency 20% less impactful
    modifiers:
      trajectory_rules:
        promotion_bonus: 1.5 # Promotions are more valuable for leaders

  specialist:
    multipliers:
      recency_score: 1.5      # Makes recency 50% more important
    modifiers:
      trajectory_rules:
        stable_reliable_bonus: 1.0 # Stability is twice as valuable

  graduate:
    # Instead of multipliers, we can override specific rules
    overrides:
      churn_risk_penalty_factor: 0.5 # Halve the penalty for churn risk
    modifiers:
      # We could also directly change a penalty value
      trajectory_rules:
        churn_risk_penalty: -1.0 # Reduced from the default -2.0
Step 3: Update the scoring_service.py
The core scoring function calculate_final_assessment needs to be updated to use this new structure.

Determine Archetype: The service would first call _get_role_archetype() on the job title to get the active archetype (e.g., 'leadership').

Load Scoring Rules: It would load the default set of rules, then "layer" the specific overrides and multipliers from the leadership archetype on top of it.

Apply Modifiers: When calculating the final score, it would apply the new logic. For example:

Instead of just final_score += recency_bonus, it would be final_score += (recency_bonus * multipliers.get('recency_score', 1.0)).

It would use the overridden promotion_bonus of 1.5 instead of the default 1.0.

3. Example Scenario: "Specialist" Role
Let's see this in action for a "Senior Cloud Architect" role, which would be classified as a Specialist.

Candidate A: credibility_score: 8.0, recency_score: 9.5 (very current skills).

Candidate B: credibility_score: 9.0, recency_score: 6.5 (stable but older skills).

Without Archetypes: Candidate B might score higher due to their better credibility.

With the "Specialist" Archetype:

The system loads the specialist rules, which include recency_score_multiplier: 1.5.

The +0.5 point bonus for high recency (recency_score > 8.0) would be effectively boosted for Candidate A: 0.5 * 1.5 = +0.75 points.

Candidate B gets no recency bonus.

The final score for Candidate A is now significantly higher, correctly reflecting that for a Specialist role, having cutting-edge skills is more important than a slightly higher credibility score.

4. UI/UX Impact
Settings Page: The "Scoring Logic" editor would need a major redesign. Instead of a single list of fields, it would likely have tabs or an accordion for each archetype (Default, Leadership, Specialist, etc.), allowing an admin to configure the overrides and multipliers for each.

Analysis Page: To make the logic transparent, it would be beneficial to add a small tag next to the job title indicating the detected archetype (e.g., "Senior Cloud Architect [Specialist Role]"). The tooltip for the final score could also state, "Calculated using the 'Specialist' scoring profile."

5. Potential Challenges
Keyword Ambiguity: A title like "Senior Lead" could match both "Specialist" and "Leadership". The archetype detection logic will need a clear order of precedence.

Configuration Complexity: The scoring_logic.yaml file becomes more complex. The UI for managing it must be very intuitive to prevent misconfiguration.

Balancing: The multipliers and overrides will need careful tuning to ensure they produce the desired outcomes without creating extreme or nonsensical scores.