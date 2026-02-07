# Migrating a Large Monorepo from Perforce to Git (Azure DevOps)

## What this was
A full-history migration of a large Perforce depot into Git (hosted in Azure DevOps), while keeping the ability to continue syncing changes during the transition period.

This was not a “copy files and call it done” migration. It involved history import, history rewrite, push-size constraints, cross-platform workflows, and (most importantly) change-management with engineers who had lived in Perforce for years.

## The problem
We needed to move a production, multi-platform codebase from Perforce to Git:

- The repository was **~100 GB** at the time of migration.
- Developers across **Windows, macOS, and Linux** worked in the **same repo**.
- There was meaningful human friction: some developers had never used anything but Perforce and were understandably skeptical about workflow changes.
- The migration had to preserve history and enable validation that the Git result matched Perforce.

## Constraints that made this hard
### 1) Large history + push limitations
Even when individual changes are small, a monorepo with long history can create push-size and history-transfer constraints.
In my case, the destination Git host had a **push size limit** that forced a strategy for large commits and/or history rewriting.

### 2) “Bad history” exists even if you cleaned Perforce
At least one problematic path had already been removed from Perforce “current state,” but still existed in history.
That meant a straight import would still carry the problem forward unless history was rewritten.

### 3) Migration is technical *and* cultural
The biggest risk was not Git commands—it was avoiding downtime and confusion:
- ensuring developers trusted that the Git repo was accurate
- providing a path for ongoing updates while people adapted

## High-level strategy
1. **Import full history** from Perforce using `git p4`.
2. Create a clean Git branch for the “publishable” history.
3. **Rewrite history** on that branch to remove problematic paths and reduce oversized history segments.
4. Push to Azure DevOps in a controlled way (batching if needed).
5. During transition, keep pulling Perforce updates and selectively applying them to the Git branch.

## Key implementation choices

### Import from Perforce (history-preserving)
The initial import used `git p4 clone` (Perforce depot path sanitized):

```bash
git p4 clone //depot/path main_repo
````

### History rewrite for “pushability” and hygiene

To remove problematic historical content and reduce oversized history, I used `git filter-repo` (paths sanitized):

```bash
git filter-repo \
  --path big/file1.tar.gz \
  --path big/file2.tar.gz \
  --path bad/path/.git \
  --invert-paths \
  --force
```

A critical nuance: **history rewrite breaks the ability to keep syncing with Perforce** on that same branch.
So the workflow separated:

* a branch that continued to track Perforce updates
* a branch that was rewritten and pushed to Azure DevOps

### Ongoing updates after the initial cutover

After the “clean branch” was published, subsequent Perforce updates were pulled and applied in a controlled manner:

```bash
git checkout master
git p4 rebase

git checkout main
git cherry main master   # identify what’s new
git cherry-pick <first>^..<last>
git push origin main
```

This created a workable bridge period where Perforce remained the source of truth while Git adoption ramped up.

## Validation

To ensure the migration was correct, I validated repository state between Perforce and Git using a file comparison tool (example: Beyond Compare) and a dedicated Perforce workspace.
The goal was simple: confirm the Git repo matched Perforce for equivalent snapshots.

## What I’d improve next

### Reduce repo size with large-file offloading (Git LFS)

A large portion of monorepo weight often comes from big non-text artifacts (archives, third-party drops, binaries).
If I were doing this again (or continuing optimization), I would:

* move large artifacts to a dedicated storage strategy (e.g., **Git LFS**)
* keep the main repo lean and fast for day-to-day dev workflows

## Why this belongs on my “core projects” list

This migration exercised the full stack of platform engineering skills:

* source control and history mechanics
* risk management and validation
* cross-platform workflow correctness
* change management across people and tooling
* operational planning for staged cutover
