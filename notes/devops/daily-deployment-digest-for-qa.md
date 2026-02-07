# Making Deployed Changes Visible to an Overseas QA Team

## Problem

Our QA team works overnight relative to the development team. They needed a reliable way to answer a simple but critical question:

> **Which code changes are actually built and deployed into SaaS right now?**

This was difficult because:
- QA did not have direct access to Azure DevOps.
- Even with access, it was non-trivial to determine whether a Jira ticket’s fix had actually made it into the deployed SaaS build.
- The Jira ↔ Azure DevOps integration was not sufficient for clearly answering “what is deployed,” despite showing build/deploy checkmarks.

As a result, QA often had to rely on manual clarification or incomplete signals, which slowed testing and increased uncertainty.

## Constraints

- QA did not have (and should not require) access to Azure DevOps.
- The solution needed to work across time zones with zero handoff friction.
- No new portal access should be introduced beyond what QA already had.

## Solution

I implemented a lightweight **daily deployment digest** that runs at the end of the development shift and produces a single report answering:

> “What was committed today, and what is actually deployed?”

Key characteristics:
- A scheduled pipeline runs once per day.
- The pipeline gathers commits made during the day and correlates them with build/deploy status.
- A human-readable report is generated and uploaded to an Azure Storage Account.
- QA accesses the report using **Azure Storage Explorer**, which they already used for log access.

This provided QA with a single, trustworthy source of truth each morning without requiring CI/CD access or additional permissions.

## High-level implementation

At a high level, the system:
1. Runs a pipeline at a fixed daily time.
2. Queries build and deployment results for key pipelines.
3. Collects commits within the defined time window.
4. Produces a structured text report grouped by project/component.
5. Publishes the report to blob storage, overwriting the previous daily version.

The report intentionally favors clarity over technical detail so it can be consumed directly by QA.

## Key engineering challenges

### 1) Stable report formatting

The report format needed to remain stable and readable because it was consumed directly by humans.
Maintaining section boundaries and preventing formatting drift as the report evolved required careful handling.

### 2) Reconciling transient failures

If a commit was initially associated with a failed build or deployment (for example, due to a CI/CD issue) and later succeeded, the system needed to:
- remove any lingering “FAILED” markers
- avoid permanently flagging commits that were eventually deployed

This prevented false alarms and unnecessary re-testing.

### 3) Least-privilege access

QA already had access to Azure Storage Explorer for logs.
The solution reused this existing access model and avoided introducing new portal permissions or tooling.

## Outcomes

- QA gained a clear, daily view of what was actually deployed.
- Reduced back-and-forth across time zones.
- Fewer “are we testing the right build?” questions.
- Calmer, more predictable overnight handoffs.

## Future improvement

If rebuilding this today, I would store the report as structured data (for example, **Azure Table Storage**) instead of a flat text blob.
That would simplify updates, reduce formatting complexity, and make filtering by date, component, or status much easier.
