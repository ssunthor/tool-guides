# Behavioral Analysis Predictor (BAP) - Design Document

**Version:** 1.0
**Date:** 2025-09-17
**Author(s):** Martin Kecka, Gemini AI
**Category:** Processor (P)
**Acronym:** `BAP`

---

## 1. Introduction & Purpose

The **Behavioral Analysis Predictor (BAP)** is a next-generation talent assessment module designed to provide a deep, context-aware understanding of a candidate's behavioral competencies and work styles. It serves as a modern alternative to traditional psychometric tests (e.g., SHL OPQ32), which often rely on abstract, self-reported data that can lack relevance to specific job functions.

BAP's core purpose is to move beyond asking candidates *what they think they are like* and instead evaluate *how they would act* in realistic, job-relevant situations.

The system is built on a two-pillar methodology:
* **Dynamic Situational Judgment:** Presenting candidates with open-ended, role-specific scenarios to assess their practical problem-solving and decision-making skills.
* **Evidence Synthesis:** Cross-validating the candidate's claimed behaviors against the factual evidence of their career history, derived from the **CV Analysis Processor (CAP)** module, to produce a more accurate and reliable assessment.

---

## 2. The Assessment Methodology in Detail

BAP's accuracy stems from its unique, multi-stage approach to evaluating behavior.

### 2.1. The Hybrid Assessment Model

To create the most robust and insightful behavioral profile, BAP will employ a two-level hybrid assessment model. This approach combines the efficiency of quantitative, multiple-choice questions with the depth of qualitative, open-ended scenarios.

---
#### **Level 1: Broad Competency Screening (Multiple-Choice SJT)**

This is the first stage of the assessment, designed for speed and broad coverage.

* **Format:** The candidate is presented with **3-4 multiple-choice Situational Judgment Test (SJT)** questions.
* **Structure:** Each question includes a realistic work scenario followed by 4-5 plausible courses of action. The candidate must select the **"Most Effective"** and **"Least Effective"** options.
* **Purpose:**
    * To quickly and efficiently gather quantitative data on a candidate's judgment across several core competencies.
    * To provide a standardized baseline score that can be easily compared across all candidates.
    * To identify potential areas of weakness or unique strength that warrant a deeper look.

---
#### **Level 2: Deep Dive Analysis (Dynamic Open-Ended Scenario)**

This is the second stage, designed to capture the candidate's unique thought process and communication style.

* **Format:** The candidate is presented with **one complex, open-ended scenario** and is asked to write a free-text response explaining their approach.
* **Dynamic Selection (Advanced Feature):** The AI can dynamically select this scenario based on the candidate's performance in Level 1. For example, if a candidate scored lowest on the "Stakeholder Influence" SJT, the AI will present them with an open-ended scenario focused specifically on that competency.
* **Purpose:**
    * To gather rich, qualitative data on the candidate's problem-solving skills, reasoning, and written communication style.
    * To see *how* a candidate thinks, not just *what* they would choose from a list.
    * To provide the AI with a "full story" to analyze using the detailed behavioral rubrics and cross-validate against the CAP module data.

This hybrid model ensures the assessment is both efficient (respecting the candidate's time) and deeply insightful, giving you a far more complete picture of a candidate's potential than the OPQ32's purely standardized approach.

### 2.2. Pillar 2: AI-Powered Analysis & Scoring

Once a candidate submits their responses, the AI analyzes the unstructured text to produce a structured, quantitative assessment.

* **Behavioral Scoring Rubrics:** For each competency, the AI uses a pre-defined, multi-dimensional rubric to score the candidate's response. This ensures consistency and objectivity across all candidates.

**Example Scoring Rubric for "Proactive Communication":**
The AI would analyze the candidate's story and score it against these dimensions:
+---------------------------+----------------------------------------------------------------------------+
| Dimension                 | High-Score Indicators (Evidence of...)                                     |
+---------------------------+----------------------------------------------------------------------------+
| Timeliness                | Acknowledging the need to communicate immediately, not waiting             |
|                           | for problems to escalate.                                                  |
+---------------------------+----------------------------------------------------------------------------+
| Audience Awareness        | Tailoring the message differently for their own team vs. senior            |
|                           | stakeholders.                                                              |
+---------------------------+----------------------------------------------------------------------------+
| Clarity & Transparency    | Clearly stating the problem, its (limited) impact, and the proposed        |
|                           | solution without hiding information.                                       |
+---------------------------+----------------------------------------------------------------------------+
| Solution-Orientation      | Focusing the communication on the plan forward, not just on the            |
|                           | problem itself.                                                            |
+---------------------------+----------------------------------------------------------------------------+

* **Rationale Generation:** The AI doesn't just provide a score; it provides a concise, human-readable rationale explaining *why* the candidate received that score, quoting snippets of their response as evidence.

### 2.3. Pillar 3: Evidence Synthesis & Cross-Validation with CAP

This is the final and most critical step for ensuring accuracy. BAP's findings are not treated as absolute truth but are cross-validated against the candidate's verified history from the CAP module.

* **API Communication:** The BAP service makes a request to the CAP service, sending its preliminary behavioral findings and requesting relevant historical data for the candidate.
* **Alignment Check:** The system compares the *claimed behavior* (from BAP) with the *demonstrated history* (from CAP).
    * **Alignment Example (✅ High Confidence):** BAP finds the candidate has excellent "Stakeholder Influence" skills based on their scenario response. CAP data confirms the candidate's CV shows a clear progression from technical roles to team lead to project manager, providing strong evidence of their ability to influence others.
    * **Misalignment Example (⚠️ Low Confidence):** BAP finds the candidate gives a great theoretical answer on "Managing Ambiguity." However, CAP data shows their entire work history consists of junior roles in highly structured, process-driven environments. The system flags this as a potential risk to explore in an interview.

---

## 3. User Workflow & Experience

1.  **Step 1: Define Behavioral Profile:** When creating a job, the hiring manager navigates to the BAP tab. They can manually add key competencies or click "🤖 **Suggest Competencies**" to have the AI analyze the job description and propose the 3-5 most relevant ones.
2.  **Step 2: Send Assessment:** The hiring manager sends a unique assessment link to the candidate.
3.  **Step 3: Candidate Experience:** The candidate completes a short, engaging assessment (typically 3-5 open-ended questions), which feels more like a practical case study than a traditional personality test.
4.  **Step 4: Review BAP Report:** Once completed, the candidate appears on the BAP dashboard. The hiring manager can click to view the full **Behavioral Competency Profile**, which includes scores, AI-generated rationales, and the critical cross-validation flags from CAP.

---

## 4. The Final Report: Behavioral Competency Profile

The output is a clear, actionable report designed for hiring managers.

+---------------------------------+--------------------------------------------------------------------------------+
| Component                       | Description                                                                    |
+---------------------------------+--------------------------------------------------------------------------------+
| Overall BAP Score               | An aggregated score (1-10) representing the candidate's overall                |
|                                 | performance in the assessment.                                                 |
+---------------------------------+--------------------------------------------------------------------------------+
| Competency Breakdown            | A series of bar charts showing the candidate's score for each specific         |
|                                 | competency (e.g., Resilience: 8.2/10, Communication: 9.1/10).                  |
+---------------------------------+--------------------------------------------------------------------------------+
| AI Rationale & Evidence         | For each competency, a short summary of the AI's reasoning,                    |
|                                 | supported by direct quotes from the candidate's answer.                        |
+---------------------------------+--------------------------------------------------------------------------------+
| CAP Cross-Validation Flag       | A clear visual indicator (✅ Supported, ⚠️ Mismatch, ⚪ Neutral)              |
|                                 | showing the degree of alignment between the BAP findings and the               |
|                                 | candidate's CV analysis.                                                       |
+---------------------------------+--------------------------------------------------------------------------------+
| Synthesized Interview Questions | AI-generated questions designed to probe any identified mismatches             |
|                                 | or further explore areas of strength (e.g., "Your response on                  |
|                                 | handling ambiguity was very insightful. Can you provide an example from        |
|                                 | your time at [Previous Company] where you had to apply that skill?").          |
+---------------------------------+--------------------------------------------------------------------------------+
---

## 5. Architecture & Integration (High-Level)

* **Service Type:** (P) Processor Module with a user-facing UI and a backend analysis service.
* **Interacts with:**
    * **LLM Service (AI Factory):** Relies on `gemma3:27b` for scenario generation and response analysis.
    * **Shared PostgreSQL Database:** For storing job competencies, candidate responses, and final reports.
    * **CAP Service (via gRPC/API):** For the critical Evidence Synthesis step, sending requests for CV data and receiving structured analysis in return.

---

## 6. Future Enhancements

* **Corporate Competency Library:** Allow organizations to create and save a standardized library of behavioral competencies for different job families (e.g., Leadership, Sales, Engineering) to ensure consistency across the company.
* **Video Response Analysis:** Evolve the input method to allow candidates to record short video responses. The AI would analyze the audio transcript and potentially sentiment/tone to add another layer of data.
* **Predictive Performance Analytics:** After 12-18 months of data collection, build a model to correlate BAP assessment scores with on-the-job performance reviews, allowing the system to continuously fine-tune its scoring rubrics for even higher predictive accuracy.