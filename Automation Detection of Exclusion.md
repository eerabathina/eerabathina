# Automation Detection of Exclusion/FP
Challenges:
Detection of exclusion/false positive (FP)
Identification of IN-PROGRESS tasks.
Consolidation of data from various sources
Information Sources:
Bigquery Update -- Provides updated data from BigQuery.       
Terraform run -- Logs related to Terraform execution.
Snow Report -- Updates retrieved from the ServiceNow (SNOW) platform
Proposed Flow:
Extract Report from SNOW – through automation
Extract automation details from GCP logging.
Eg. Unique ID
BigQuery Result
Extract TFC run id(s)
Connect to TFC and query extracted run id.
Compare results from all sources.
Validation Logic: Write logic to determine whether the task is an exclusion or a false positive.
Consolidation: Concatenate all results into a single sheet for comprehensive 
analysis.
Different Types of Exclusions:
Whitelisting:
Self Service Tag:
Branch Unstable:
Missing SA:
Multiple Tasks:
Unsupported Role:
Different Types of FP:
Completed successfully but updated failure.
Completed successfully but updated IN Progress.
Completed successfully but timeout occurred.
Exclusion Types:
Name
Automation Possibility
Whitelisting
Yes
Self-serve tag is true 
Yes
Branch unstable
Running tasks before SA creation
Multiple tasks running parallel
This document outlines the core challenges and proposed solutions for handling automation detection related to exclusions and false positives, ensuring streamlined and effective processes.
# Plan of Action
Below is a clear, actionable Plan of Action (PoA) for RBAC Automation Failures covering rootcause isolation, exclusion identification, remediation, and prevention.Aligned with real failure patterns (TFC run success but SNOW status wrong, timeouts, branch divergence, unsupported roles, etc.) and kept it practical for GCP + Terraform Cloud + ServiceNow automation.
Plan of Action – RBAC Automation Failures
1. Problem Statement
RBAC automation is intermittently failing or reporting incorrect states in ServiceNow (SNOW) despite Terraform Cloud (TFC) runs completing. Failures fall into multiple categories:
Status mismatch (Completed but marked Failed / In Progress)
Timeouts
Environmental and design constraints (branch divergence, unsupported roles, missing service accounts)
Operational conflicts (parallel runs, exclusions not detected)
2. High-Level Objectives
Accurately classify failures (Real failure vs Expected exclusion)
Auto-detect exclusions early to avoid false failures
Align TFC run state with SNOW ticket state
Reduce rework and manual reconciliation
Prevent recurrence via guardrails
3. Failure Classification Framework
A. Status Mismatch Issues
Scenario
Likely Cause
Action
Completed successfully but SNOW shows Failed
Callback/update from automation to SNOW failed or timed out
Introduce post-run reconciliation job
Completed successfully but SNOW shows In Progress
Final status update not triggered
Add mandatory final-state sync check in automation
Completed successfully but Timeout occurred
TFC finished after SNOW timeout threshold
Update timeout to match execution time or correct logic flaw.
4. Exclusion Identification Logic (Critical)
Automation must not fail tickets when exclusions are valid.
A. Exclusion Decision Matrix
Condition
Detection Method
Classification
Action
Whitelisting present
Parse SNOW ticket description/variables
Exclusion
Mark ticket as Completed – Excluded
Self Service Tag present
Check request metadata / tags
Exclusion
Mark ticket excluded – irrespective of result
Branch unstable (dev ≠ main)
Git divergence check
Non-actionable
Fail fast with actionable error. It is already in place ??
Missing Service Account
GCP IAM / SA existence check
Real failure
Fail with remediation steps
Parallel tasks running
Locking / run-id check
Retryable
Queue or delay execution
Unsupported GCP Role
Static role allowlist
Exclusion
Auto-reject with reason
5. Detailed Actions by Challenge
5.1 TFC Run Completed but SNOW Ticket Incorrect
Root Cause
SNOW status update depends on async callbacks or job completion signals that may fail.
Actions
Introduce Post-TFC Reconciliation Job 
Poll TFC run status using Run ID
If applied, force-update SNOW ticket to Completed
Store Run ID ↔ SNOW Ticket ID mapping
Implement idempotent status updates (safe to retry)
 Outcome: Eliminates false “Failed” and “In Progress” states
5.2 Completed but Timeout Occurred
Root Cause
SNOW timeout < actual TFC apply duration
Actions
Increase SNOW timeout threshold or correct the logic for updating SNOW
Or correct logic for timeout error.
Mark ticket as Execution Ongoing – Awaiting Terraform Completion
 Outcome: Timeout no longer treated as failure
5.3 Branch Diverged (dev vs main)
(Possible fix is already in place)
Root Cause
Automation assumes dev and main are aligned
Actions
Pre-flight Git Check 
Detect divergence (ahead & behind commits)
Fail before TFC run starts
Update SNOW with: 
“Branch unstable – manual reconciliation required”
Commit counts (ahead/behind)
 Outcome: Prevents invalid runs and noisy failures
5.4 Missing Service Account(We have fix for it already)
Root Cause
RBAC assignment attempted before SA creation
Actions
Pre-check: 
Validate SA existence in target project
If missing: 
Auto-fail with clear remediation message
Optionally auto-create SA if policy allows
 Outcome: Clear ownership and faster resolution
5.5 Multiple Tasks Running in Parallel
Root Cause
Concurrent RBAC runs on same project/resource
Actions
Introduce distributed lock (project + role scope)
If lock exists: 
Queue request OR
Retry with backoff
Update SNOW as Waiting – Another RBAC task in progress
 Outcome: Prevents race conditions and partial failures
5.6 Unsupported GCP Role
Root Cause
Role not supported by automation or Terraform module
Actions
Maintain static allowlist of supported roles
Validate role before TFC run
If unsupported: 
Auto-reject ticket
Route to manual GCP Ops queue
 Outcome: Zero wasted TFC runs
6. Unified Pre-Flight Validation (Key Improvement)
Create a single pre-flight validation stage executed before any TFC run:
Pre-Flight Checks
 Exclusion tags Branch stability Service account existence Supported role Parallel execution lock
If any check fails → do NOT start TFC run
7. Improved Ticket Status Model (SNOW)
State
Meaning
Validation Failed
Pre-flight failed (no infra change attempted)
Excluded
Whitelisting / Self-service
Execution In Progress
TFC running
Completed
TFC applied successfully
Completed – Excluded
Expected exclusion
Failed – Action Required
Genuine failure
8. KPIs to Track
% tickets auto-closed as Excluded
% false failures eliminated
Avg reconciliation time
Timeout-related failures
Manual interventions per week
9. Immediate Next Steps
Implement pre-flight validation framework
Add Git divergence detection
Introduce post-TFC reconciliation job
Update SNOW status checks
Roll out role allowlist + SA existence checks (we already have it)

|Name|Automation Possibility|
|---|---|
|Whitelisting|Yes|
|Self-serve tag is true|Yes|
|Branch unstable||
|Running tasks before SA creation||
|Multiple tasks running parallel||


|Scenario|Likely Cause|Action|
|---|---|---|
|Completed successfully but SNOW shows Failed|Callback/update from automation to SNOW failed or timed out|Introduce post-run reconciliation job|
|Completed successfully but SNOW shows In Progress|Final status update not triggered|Add mandatory final-state sync check in automation|
|Completed successfully but Timeout occurred|TFC finished after SNOW timeout threshold|Update timeout to match execution time or correct logic flaw.|


|Condition|Detection Method|Classification|Action|
|---|---|---|---|
|Whitelisting present|Parse SNOW ticket description/variables|Exclusion|Mark ticket as Completed – Excluded|
|Self Service Tag present|Check request metadata / tags|Exclusion|Mark ticket excluded – irrespective of result|
|Branch unstable (dev ≠ main)|Git divergence check|Non-actionable|Fail fast with actionable error. It is already in place ??|
|Missing Service Account|GCP IAM / SA existence check|Real failure|Fail with remediation steps|
|Parallel tasks running|Locking / run-id check|Retryable|Queue or delay execution|
|Unsupported GCP Role|Static role allowlist|Exclusion|Auto-reject with reason|


|State|Meaning|
|---|---|
|Validation Failed|Pre-flight failed (no infra change attempted)|
|Excluded|Whitelisting / Self-service|
|Execution In Progress|TFC running|
|Completed|TFC applied successfully|
|Completed – Excluded|Expected exclusion|
|Failed – Action Required|Genuine failure|
