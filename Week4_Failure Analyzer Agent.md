Failure Analyzer Agent
Technical Architecture & System Design Document
1. Objective

Design and implement a Failure Analyzer Agent that runs inside the same Azure DevOps pipeline job immediately after Playwright UI tests execution.

The agent must:

Read failure artifacts

Pre-classify failure using rule engine

Use Azure OpenAI for semantic reasoning

Identify root cause

Summarize clearly

Suggest next action

Update pipeline summary

Send Microsoft Teams notification

Send Email

Create Azure DevOps Work Item (Bug) for every failed test

Operate statelessly per pipeline run

Process each failed test individually

2. Confirmed Technology Stack
Component	Technology
Language	Python
Test Framework	Playwright (UI)
Dependency Management	pipenv
CI/CD	Azure DevOps
LLM	Azure OpenAI
Packaging	Python CLI tool
State	Stateless per pipeline run
3. High-Level Architecture
Azure DevOps Pipeline
        │
        ├── Step 1: Execute Playwright Tests
        │
        ├── Step 2: Publish Test Artifacts
        │
        └── Step 3: Run Failure Analyzer CLI
                        │
                        ├── Artifact Loader
                        ├── Failure Extractor (JUnit Parser)
                        ├── Rule-Based Classifier
                        ├── Context Builder
                        ├── Azure OpenAI Root Cause Engine
                        ├── Recommendation Generator
                        ├── Work Item Creator
                        ├── Notification Dispatcher
                        └── Pipeline Summary Writer
4. System Design Phases
Phase 1 – Artifact Ingestion
Inputs (per failed test):

Console logs

Network logs

Screenshot path

JUnit XML report

Components:
4.1 JUnit Failure Extractor

Parse JUnit XML

Identify failed test cases

Extract:

Test name

Suite name

Error message

Stack trace

Duration

Output:

{
  "test_id": "...",
  "error_message": "...",
  "stack_trace": "...",
  "status": "FAILED"
}
Phase 2 – Rule-Based Pre-Classification Engine

Purpose:
Fast deterministic classification before invoking LLM.

5.1 Failure Categories
Category	Detection Strategy
Assertion Failure	Regex: AssertionError
Timeout	Regex: TimeoutError, waiting for
Locator Not Found	NoSuchElement, locator
Network Failure	HTTP 5xx, 4xx
Page Crash	page crashed, Target closed
Environment Failure	browser not found, infra logs

Output:

{
  "pre_classification": "Timeout",
  "confidence": 0.82
}
Phase 3 – Context Builder

Construct structured LLM input including:

Test metadata

Failure message

Stack trace

Extracted relevant console logs

Relevant network failures

Pre-classification result

Output format:

{
  "test_name": "...",
  "failure_type_rule": "...",
  "error": "...",
  "network_errors": [...],
  "console_snippet": "...",
  "screenshot_reference": "path"
}

Screenshots are not passed as binary — only described contextually.

Phase 4 – Azure OpenAI Root Cause Engine
Model: Azure OpenAI (GPT-4 class model)
Prompt Structure
SYSTEM:
You are a senior QA automation architect specialized in Playwright UI failures.

USER:
Analyze the following failed test.

Pre-classification: {rule_class}
Failure message: {error_message}
Stack trace: {stack_trace}
Network errors: {network_summary}
Console logs: {console_snippet}

Tasks:
1. Identify the most probable root cause
2. Confirm or override pre-classification
3. Provide confidence score (0-100)
4. Suggest next action
5. Write concise bug summary
6. Provide developer-friendly reproduction hint
Expected LLM Output (Structured JSON)
{
  "final_failure_type": "...",
  "root_cause": "...",
  "confidence_score": 87,
  "next_action": "...",
  "bug_summary": "...",
  "reproduction_hint": "..."
}
5. Per-Test Processing Flow

For each failed test:

FOR each failed test:
    Parse JUnit
    Extract logs
    Run rule engine
    Build context
    Call Azure OpenAI
    Generate structured response
    Create Work Item
    Append to pipeline summary
    Add to notification payload
END
6. Azure DevOps Work Item Creation
Type: Bug

Created for every failed test

Bug Fields:
Field	Value
Title	[Auto][UI Failure] {Test Name}
Description	LLM Generated Summary
Repro Steps	Reproduction Hint
Root Cause	Root Cause Explanation
Failure Type	Classified Type
Confidence	Score

Artifacts are NOT attached (as confirmed requirement).

Integration via:

Azure DevOps REST API

Personal Access Token (PAT) or Managed Identity

7. Notification System
7.1 Pipeline Summary Update

Use Azure DevOps logging commands:

##vso[task.uploadsummary]summary.md

Generate markdown:

# Failure Analyzer Report

| Test | Failure Type | Confidence | Bug ID |
|------|--------------|------------|--------|
7.2 Microsoft Teams Notification

Payload:

{
  "title": "Playwright Failures Detected",
  "failures": [
    {
      "test": "...",
      "type": "...",
      "bug_id": "...",
      "confidence": 87
    }
  ]
}

Integration via Teams Webhook.

7.3 Email Notification

Summary email:

Subject:

[Pipeline Failed] UI Test Failures - {Build Number}

Body:

Total failures

Top root causes

Bug links

8. CLI Tool Design
Command
pipenv run python failure_analyzer.py \
  --junit path/results.xml \
  --console logs/console.log \
  --network logs/network.json \
  --screenshots screenshots/
Modules
failure_analyzer/
│
├── cli.py
├── junit_parser.py
├── log_extractor.py
├── rule_engine.py
├── context_builder.py
├── llm_client.py
├── bug_creator.py
├── notifier.py
├── pipeline_summary.py
└── models.py
9. Security & Secrets

Secrets stored in Azure DevOps secure variables:

AZURE_OPENAI_ENDPOINT

AZURE_OPENAI_KEY

AZURE_DEVOPS_PAT

TEAMS_WEBHOOK_URL

SMTP_CREDENTIALS

No secrets stored in code.

10. Non-Functional Requirements
Attribute	Strategy
Scalability	Process tests sequentially (stateless)
Cost Control	Hybrid model reduces LLM calls
Determinism	Rule engine first pass
Reliability	Retry LLM on transient failure
Observability	Internal debug logging
Idempotency	Unique bug per failure
11. Future Enhancements (Not in Phase 1)

Flakiness detection

Historical correlation

Duplicate bug detection

Screenshot vision analysis

Failure clustering

Auto-assignment of bug

Root cause confidence calibration

12. Agent Prompt Definition (.md for Copilot)
# Failure Analyzer Agent

## Role
You are a senior QA automation architect specialized in analyzing Playwright UI test failures.

## Objectives
For each failed test:
- Identify root cause
- Confirm or override rule-based classification
- Provide confidence score
- Suggest next action
- Write concise bug-ready summary
- Provide reproduction hint

## Constraints
- Be deterministic
- Do not hallucinate missing logs
- Use only provided context
- Output strict JSON

## Output Schema
{
  "final_failure_type": "",
  "root_cause": "",
  "confidence_score": 0,
  "next_action": "",
  "bug_summary": "",
  "reproduction_hint": ""
}
13. End-to-End Flow Summary

Playwright runs

Failures detected

CLI triggered

JUnit parsed

Rule classification

Azure OpenAI reasoning

Bug created (per failure)

Pipeline summary updated

Teams + Email sent

Pipeline completes

Final Outcome

You now have:

Enterprise-ready architecture

Agentic phased reasoning design

Deterministic + LLM hybrid model

Azure-native integration

Copilot-ready prompt specification

Scalable stateless system

If you would like next, we can design:

Detailed sequence diagram

Sample Azure DevOps YAML integration

Production-ready prompt engineering refinement

Cost estimation model for Azure OpenAI usage