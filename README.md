![Infrastructure & Deployment Notes](GithubReadmeBanner.png)

# Infrastructure & Deployment Notes

This repository contains selected notes, patterns, and small utilities derived from
real-world work on cloud infrastructure, Kubernetes, identity, and SaaS deployments.

The focus is not on polished libraries or demo projects, but on:
- understanding system behavior
- operational decision-making
- deployment reliability
- configuration drift, cost, and security trade-offs

Most examples are Azure-centric, with general principles applicable across cloud platforms.

---

## Featured notes

These are a small number of representative pieces that capture how I approach
infrastructure, deployment, and operational problems:

- **Configuration drift detection with Bicep and what-if**  
  [`notes/iac/bicep-drift-detection-with-whatif.md`](notes/iac/bicep-drift-detection-with-whatif.md)  
  Detecting unintended configuration changes in production using Azure Resource Manager what-if
  and a daily verification pipeline.

- **Migrating a large monorepo from Perforce to Git**  
  [`notes/version-control/perforce-to-git-migration.md`](notes/version-control/perforce-to-git-migration.md)  
  A history-preserving migration of a ~100 GB, cross-platform repository, including
  history rewriting, validation, and staged cutover.

- **UTF-16 files, line endings, and Git in a cross-platform repo**  
  [`notes/version-control/utf16-eol-in-a-cross-platform-repo.md`](notes/version-control/utf16-eol-in-a-cross-platform-repo.md)  
  Handling UTF-16 files safely in Git using explicit attributes to avoid silent corruption
  and cross-platform EOL issues.

---

Additional notes are organized by area under the `notes/` directory.
