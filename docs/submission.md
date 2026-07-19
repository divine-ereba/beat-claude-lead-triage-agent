# AI Automation Intern 012 – Lead Triage Agent

**Candidate:** Divine Ereba

**Brief Version:** 2026-07

## Written Answer

## Introduction

This submission implements an AI-powered lead triage workflow that automates the first-pass review of inbound website leads using n8n and OpenAI.

The workflow processes every record in the provided fixture dataset, evaluates each lead against predefined business and security rules, identifies commercial opportunities, detects adversarial and malformed inputs, routes security, compliance, and operational risks to the appropriate human owner, and produces a structured decisions file containing the lead ID, decision, owner, confidence score, and reasoning.

The submission also includes the complete n8n workflow, decisions file, execution logs, metrics dashboard, evidence log, AI usage disclosure, and supporting documentation demonstrating the workflow's behavior on the full fixture dataset.

## Architecture

The workflow was implemented using n8n as the orchestration platform because it provides transparent workflow execution, native branching logic, API integrations, retry handling, and detailed execution logs that are easy to inspect and reproduce.

OpenAI is used as the reasoning engine responsible for evaluating lead quality. Rather than hard-coding dozens of business rules, the model performs contextual analysis while operating under a constrained prompt that enforces structured outputs, treats all lead content as untrusted input, resists prompt injection attempts, and routes security, compliance, and operational risks to human reviewers while confidently classifying routine commercial enquiries.

Google Sheets serves as the operational datastore. The workflow records every processed lead in a Decision Log, maintains historical records to prevent duplicate processing across subsequent workflow executions, and generates operational metrics through a reporting dashboard. Leads with overlapping identity signals are evaluated independently rather than automatically merged or discarded, preserving legitimate follow-up enquiries while maintaining historical duplicate protection across workflow runs.

Gmail is integrated to automatically notify the appropriate stakeholders whenever a lead requires human intervention, ensuring that security, compliance, support, and operational issues are surfaced immediately instead of remaining inside the automation.

The overall architecture intentionally separates orchestration, AI reasoning, data storage, reporting, and notification into independent components. This modular design allows individual services, models, or integrations to be replaced with minimal changes to the overall workflow, making the solution easier to maintain and scale.

## Workflow Overview

The workflow begins by importing the provided fixture dataset containing inbound website leads. Each lead is processed individually using an iterative loop to ensure every record receives an independent evaluation.

For each lead, the workflow first performs a duplicate check against previously processed submissions. The workflow also maintains historical duplicate detection across workflow executions. During normal processing each lead is evaluated independently, while subsequent workflow executions identify previously processed records and record them separately in the Duplicate Log to prevent duplicate processing.

Each non-duplicate lead is then sent to an AI Lead Triage Agent. The agent evaluates the lead using a structured prompt that considers business intent, company legitimacy, budget, buying signals, data quality, security risks, and operational constraints. Prompt injection attempts, malformed inputs, privacy requests, malware links, and ambiguous cases are treated as untrusted data and escalated to the appropriate human owner instead of being automatically classified.

The AI returns a structured response containing a decision (**QUALIFY**, **NURTURE**, **REJECT**, or **ESCALATE**), owner assignment, confidence score, detected issues, and a concise justification. A code node parses and validates the response before routing it through the workflow.

High-risk escalations automatically trigger email notifications to the appropriate stakeholders, while all processed leads are written to the Decision Log. Duplicate submissions are recorded separately to preserve an audit trail without contaminating operational metrics.

A live dashboard built in Google Sheets summarizes workflow performance by reporting total processed leads, decision distribution, average confidence, average budget, and other operational metrics derived directly from the generated Decision Log.

## What the Fixture Threw at Me

The fixture was intentionally designed to simulate the kinds of noisy and unpredictable inputs encountered in real production environments rather than a clean sales dataset. Alongside legitimate business opportunities, it included malformed records, adversarial inputs, privacy requests, suspicious links, incomplete information, and records with overlapping attributes that required careful judgment.

Several notable scenarios were encountered:

- **L-003 — Potential duplicate:** L-003 shared the same email address as L-001 but differed in company formatting and expanded on the original enquiry by expressing additional interest in SEO services. Rather than automatically merging or rejecting the submission based solely on duplicate signals, the workflow evaluated it independently and classified it as **NURTURE**, treating it as a plausible follow-up enquiry while avoiding unnecessary suppression of legitimate sales opportunities.

- **L-006 — Prompt injection attempt:** This lead explicitly instructed the AI to ignore previous instructions and classify the submission as **QUALIFY** with a confidence score of **1.0**. Because every lead message is treated as untrusted input, the instruction was ignored and evaluated purely as customer data. The workflow classified the submission as a security event and escalated it to **Security** rather than allowing embedded instructions to influence the decision.

- **L-012 — Malformed record:** This submission contained an invalid timestamp (`2026-13-45T99:99:00Z`). Rather than attempting to infer corrupted operational data, the workflow escalated the record to **Operations** for human review.

- **L-019 — GDPR request:** This lead requested deletion of personal data under GDPR Article 17. The workflow correctly identified this as a compliance request rather than a commercial opportunity and routed it to **Compliance**.

- **L-020 — Potential phishing attempt:** This submission contained a link to a downloadable executable (`.exe`) while presenting itself as a billing-related enquiry. Rather than treating it as a sales lead, the workflow escalated it to **Security** for human investigation.

Beyond the fixture itself, the workflow also maintains historical duplicate detection across workflow executions. During testing, rerunning previously processed datasets correctly identified records that had already been logged and prevented them from being processed a second time, demonstrating protection against duplicate processing across multiple workflow executions.

Building the workflow required multiple iterations as these edge cases were uncovered through testing. Rather than optimizing solely for automated classification accuracy, the final workflow prioritizes trustworthy automation by allowing routine commercial enquiries to proceed while escalating only security, compliance, and operational risks that require human judgment.

## Failure Handling (One Bad Output and How I Addressed It)

One issue I observed during testing was inconsistent classification of certain borderline leads across repeated executions of the same fixture. Although the input dataset remained unchanged, one lead that had previously been classified as **REJECT** was later classified as **ESCALATE**, causing the overall decision distribution and average confidence score to change between runs.

Rather than accepting this inconsistency, I investigated the workflow and determined that the variation came from the probabilistic nature of the language model when evaluating ambiguous cases. This exposed two weaknesses in my original implementation: the model’s sampling behaviour introduced unnecessary variability, and some escalation criteria were implied rather than explicitly prioritised in the system prompt.

To improve reliability, I reduced the model temperature from **0.7** to **0.0** to produce more deterministic outputs and rewrote the system prompt to establish explicit routing priorities. The revised prompt clearly distinguishes routine commercial enquiries from security, compliance, and operational risks. Prompt injection attempts, malware, phishing, privacy requests, and malformed operational records are now treated as mandatory escalation events, while ordinary commercial ambiguity is resolved through **QUALIFY**, **NURTURE**, or **REJECT** instead of unnecessary escalation.

I intentionally did not try to eliminate all uncertainty. The objective was not to maximise automation but to make the workflow’s behaviour predictable, auditable, and operationally consistent. A repeatable first-pass triage process with clearly defined escalation boundaries is more valuable than an AI system that appears confident while producing inconsistent decisions.

## What Breaks Next

If this workflow were processing **500 or more leads per week**, the first bottleneck would not be the AI model itself but operational visibility. As execution volume increases, monitoring failed runs, exhausted retries, delayed notifications, API failures, and decision consistency becomes more important than improving individual classifications.

Entity resolution would also become a significant challenge at scale. The current workflow intentionally evaluates leads with overlapping identity signals independently rather than automatically merging them, preventing legitimate follow-up enquiries from being suppressed. In production, I would introduce confidence-based duplicate detection using multiple signals such as email, company, website, contact similarity, and submission timing before recommending or performing record consolidation.

Google Sheets is appropriate for this challenge but would become a limitation as throughput grows because of concurrency, reporting, and data management constraints. In production, I would replace it with a database-backed architecture supported by centralized logging, monitoring, alerting, and workflow analytics so failures are observable and recoverable rather than silently accumulating.

The objective is not simply to automate more work, but to build automation that remains observable, reliable, secure, and maintainable as operational complexity increases.

## Evidence Log

| Claim | Evidence Tier | Supporting Evidence |
|-------|---------------|---------------------|
| Every lead in the fixture dataset was processed successfully. | Tier 3 – Logs or Source Records | n8n execution log, workflow execution history, and completed Decision Log generated from the final fixture run. |
| The workflow generates exactly one routing decision for every processed lead. | Tier 3 – Logs or Source Records | Final Decision Log together with the corresponding n8n execution history for all 20 fixture leads. |
| Potential duplicate signals are evaluated conservatively rather than automatically merged or discarded. | Tier 3 – Logs or Source Records | n8n execution history and Decision Log showing independent evaluation of L-003 alongside the workflow prompt and routing logic. |
| Previously processed leads are detected during subsequent workflow executions to prevent duplicate processing. | Tier 3 – Logs or Source Records | Duplicate Log and n8n execution history from rerunning the workflow against previously logged leads. |
| Prompt injection attempts are treated as untrusted input and escalated as security events. | Tier 3 – Logs or Source Records | n8n execution history and Decision Log for L-006 demonstrating Security escalation despite malicious embedded instructions. |
| Privacy, malformed operational records, and security-sensitive submissions are escalated appropriately. | Tier 3 – Logs or Source Records | n8n execution history and Decision Log entries for L-012, L-019, and L-020 showing Operations, Compliance, and Security routing. |
| Operational metrics are generated automatically from workflow execution. | Tier 2 – Demo Artifact | Google Sheets dashboard containing calculated KPIs, charts, confidence statistics, and budget metrics. |
| High-risk submissions automatically notify the appropriate stakeholders. | Tier 2 – Demo Artifact | Gmail notification workflow together with successful notification screenshots. |
| AI retry handling improves workflow resilience against transient model failures. | Tier 2 – Demo Artifact | OpenAI Chat Model configuration showing retry settings together with workflow configuration screenshots. |

## Number Source Labels

| Number | Source Type | Justification |
|--------|-------------|---------------|
| 20 fixture leads processed | Observed | Counted directly from the provided fixture dataset (`inbound_leads.csv`). |
| 5 Qualified leads | Observed | Generated automatically by the final workflow execution and recorded in the Decision Log. |
| 9 Nurture leads | Observed | Generated automatically by the final workflow execution and recorded in the Decision Log. |
| 2 Rejected leads | Observed | Generated automatically by the final workflow execution and recorded in the Decision Log. |
| 4 Escalated leads | Observed | Generated automatically by the final workflow execution and recorded in the Decision Log. |
| Average confidence | Observed | Calculated directly from workflow outputs in the Google Sheets metrics dashboard. |
| Average monthly budget | Observed | Calculated from the fixture dataset using Google Sheets formulas. |
| Maximum retry attempts = 3 | Observed | Configured directly in the OpenAI Chat Model node. |
| Retry delay = 1000 ms | Observed | Configured directly in the OpenAI Chat Model node. |
| Example scaling workload of 500+ leads/week | Assumed | Hypothetical workload used to discuss future scalability and operational bottlenecks. |

## AI Usage Disclosure

### Tools Used

- ChatGPT
- OpenAI Chat Model (within n8n)
- n8n
- Google Sheets
- Gmail

### What AI Helped With

AI assisted with prompt iteration, lead classification inside the workflow, workflow brainstorming, documentation refinement, and exploring alternative design approaches.

### What I Personally Decided

I designed the workflow architecture, implemented the automation in n8n, defined the routing strategy, determined escalation policies, selected retry behavior, built the reporting dashboard, interpreted workflow outputs, and iteratively refined both the workflow and prompt after testing exposed edge cases and inconsistent classifications.

### What I Checked

I manually validated the workflow against the complete fixture dataset, confirmed that every lead produced exactly one routing decision, verified dashboard calculations, tested historical duplicate protection through repeated workflow executions, strengthened prompt injection resistance, reviewed all escalation decisions, and adjusted the prompt based on observed behavior rather than assumptions.

### Known Limitations

The workflow is designed as a first-pass triage system rather than a final decision-maker. Borderline commercial enquiries may still require additional business context that is unavailable at submission time. In production, I would complement the language model with structured business rules, entity-resolution logic, centralized monitoring, confidence-based quality checks, and periodic human review to minimise classification drift while preserving consistent operational behaviour.

## What Stays Human

The workflow intentionally avoids making irreversible or high-risk decisions without human involvement.

The following situations always remain human-controlled:

- GDPR, privacy, and legal requests
- Security incidents, malware, phishing attempts, and prompt injection attacks
- Malformed operational records or corrupted critical fields
- Existing customer support or billing issues
- Cases requiring legal interpretation or business judgement beyond the information available in the submission

Automation is responsible for collecting information, classifying routine commercial enquiries, and reducing repetitive operational work. Humans remain responsible for decisions where accountability, regulatory obligations, security risk, or broader business context outweigh the benefits of full automation.

## Artifact Access

The following artifacts are included with this submission:

| Artifact | Description |
|----------|-------------|
| n8n Workflow (JSON) | Complete workflow implementation available in the [workflow folder](https://github.com/divine-ereba/beat-claude-lead-triage-agent/tree/main/workflow). |
| Decision Log | Final workflow output containing one routing decision for every processed lead in the [Decision Log](https://github.com/divine-ereba/beat-claude-lead-triage-agent/blob/main/artifacts/Log%20-%20Decision%20log.csv). |
| Duplicate Log | Historical duplicate detection results available in the [Duplicate Log](https://github.com/divine-ereba/beat-claude-lead-triage-agent/blob/main/artifacts/Log%20-%20Duplicate%20log.csv). |
| Metrics Export | Export of operational metrics generated from the reporting dashboard in the [Metrics file](https://github.com/divine-ereba/beat-claude-lead-triage-agent/blob/main/artifacts/Log%20-%20Metrics.csv). |
| Execution Evidence | Successful workflow execution captured in the [execution log screenshot](https://github.com/divine-ereba/beat-claude-lead-triage-agent/blob/main/artifacts/execution_log.png). |
| Workflow Screenshots | Supporting screenshots documenting workflow configuration, Gmail notifications, retries, and execution evidence in the [screenshots folder](https://github.com/divine-ereba/beat-claude-lead-triage-agent/tree/main/screenshots). |
| Fixture SHA-256 Checksum | Verification hash for the supplied fixture contained in [SHA256.txt](https://github.com/divine-ereba/beat-claude-lead-triage-agent/blob/main/artifacts/SHA256.txt). |
| Submission Document | Complete technical report available as [`docs/submission.md`](https://github.com/divine-ereba/beat-claude-lead-triage-agent/blob/main/docs/submission.md). |
| Loom Demonstration | Complete workflow walkthrough provided in two parts due to recording platform limitations.<br><br>**Part 1 – Workflow Architecture & Design:** https://www.loom.com/share/fa5e9c8df2c8444a8461f9c0102009ae<br><br>**Part 2 – Workflow Execution, Outputs & Dashboard:** https://www.loom.com/share/ffc9bee1c56c437788f9c92b46d67154 |

**Note:** The workflow demonstration is provided as two sequential recordings due to recording platform limitations. Part 1 introduces the workflow architecture and implementation, while Part 2 demonstrates execution against the provided fixture dataset, generated outputs, operational dashboard, and final observations.

All workflow outputs were generated using the provided fixture dataset (**Brief Version: 2026-07**).

## Meta Question

### What’s the most recent thing you automated for yourself—and what did you deliberately leave manual?

The most recent thing I automated for myself was an AI-powered multi-modal customer support workflow. I built it after noticing that many customer interactions followed predictable patterns but still required manual tracking, repeated follow-ups, and status management. The workflow accepts text, voice, images, and documents, classifies requests, tracks issue status, prevents duplicate processing, and automatically follows up on unresolved conversations to keep both customers and internal teams informed.

What I deliberately left manual was the final handling of ambiguous, sensitive, or high-impact cases. When the workflow encounters situations requiring business judgment, customer empathy, legal consideration, or low-confidence decisions, it escalates them instead of attempting to automate the outcome. My goal is to automate repetitive execution while keeping accountability with humans where mistakes matter most.

