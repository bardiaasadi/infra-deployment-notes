
# Detecting Azure Configuration Drift with Bicep + `az deployment group what-if`

## Problem

In production environments, configuration changes can happen for valid reasons (hotfix, CAB-approved change, emergency mitigation) and still create risk:

- an admin (or multiple admins) changes a setting and forgets to document it
- changes drift away from what the team *thinks* is deployed
- rolling back becomes stressful because the “last known good” values aren’t easy to reconstruct

This problem is less “who did it?” and more:

> **What changed — and what were the before/after values?**

## Approach

Use **Azure Resource Manager what-if** as a drift detection mechanism:

1. **Export ARM templates** for existing resources (Key Vaults, App Services, Storage, App Gateway, etc.).
2. Convert ARM → **Bicep** (VS Code makes this easy, with some cleanup required).
3. Run `what-if` against the current environment using the Bicep templates.
4. Save the resulting `what-if` output as a **baseline** in source control.
5. Run a **daily pipeline** that repeats the same `what-if`, compares output to baseline, and fails if drift is detected.
6. If the change is legitimate, update the templates/baseline so the “desired state” stays current.

This creates a low-friction workflow where drift is detected quickly and rollback is easier because the baseline captures previous values.

## Why `what-if`?

`what-if` gives a readable summary of resource/property changes without actually deploying anything. It effectively answers:

- which resources would be modified
- which properties would change
- what values would change from/to (often surfaced in the output)

This becomes a practical “before/after values” system with minimal extra tooling.

## Pipeline flow (high level)

- Pull IaC repo onto a self-hosted agent
- Execute a `group what-if` against a known resource group and Bicep file
- Store the output (JSON or text)
- Compare to baseline output
- If there’s drift → fail pipeline and alert

### Example: pipeline entrypoint (simplified)

```yaml
trigger: none
pool: SaaS-02

variables:
  - group: DevRepoUpstream
  - name: resourceGroup
    value: 'me-saas-prod'

stages:
- stage: GitPull
  jobs:
  - job: GitPull
    steps:
    - checkout: none
    - task: PowerShell@2
      inputs:
        targetType: 'inline'
        script: |
          cd E:/master
          git pull origin $(UpstreamBranch)

- stage: me_saas_prod
  dependsOn: []
  jobs:
  - job: CheckforChanges
    steps:
    - checkout: none
    - template: ../template.yml
      parameters:
        resourceGroup: $(resourceGroup)
        subfolder: 'Gateways/me-saas-prod'
````

### Example: reusable template step

```yaml
parameters:
  resourceGroup: ''
  subfolder: ''
  basePath: 'E:/master/DevOps/IaC'

steps:
- task: AzureCLI@2
  inputs:
    azureSubscription: 'IaC'
    scriptType: 'pscore'
    scriptLocation: 'inlineScript'
    inlineScript: |
      az deployment group what-if \
        --resource-group ${{ parameters.resourceGroup }} \
        --template-file ${{ parameters.basePath }}/${{ parameters.subfolder }}/base.bicep \
        --output json > ${{ parameters.basePath }}/${{ parameters.subfolder }}/new.txt
    workingDirectory: ${{ parameters.basePath }}/${{ parameters.subfolder }}
  displayName: 'Run az deployment what-if'

- script: |
    @echo off
    setlocal
    set "output="
    for /f "delims=" %%i in ('python ../../compare_files.py ${{ parameters.basePath }}/${{ parameters.subfolder }}') do (
      set "output=%%i"
      echo %%i
    )
    if "%output%" neq "no changes" (
      echo Changes detected. Failing the pipeline...
      exit /b 1
    )
  workingDirectory: ${{ parameters.basePath }}/${{ parameters.subfolder }}
  displayName: 'Compare files and check for changes'
```

## Drift detection logic (output comparison)

A lightweight Python script compares the baseline output and the latest output and reports differences. In the simplest form:

* baseline: `base.txt`
* current run: `new.txt`
* if differences exist → print them and fail pipeline
* if none → print `no changes`

```python
import difflib
import os
import sys

def normalize_line_endings(lines):
    return [line.replace('\r\n', '\n').replace('\r', '\n') for line in lines]

def compare_files(file1, file2):
    with open(file1, 'r') as f1, open(file2, 'r') as f2:
        file1_lines = normalize_line_endings(f1.readlines())
        file2_lines = normalize_line_endings(f2.readlines())

    differences = []
    diff = difflib.ndiff(file1_lines, file2_lines)

    for line in diff:
        # capture added lines, ignore some what-if formatting noise
        if line.startswith('+ '):
            candidate = line[2:].strip()
            if not (
                candidate.startswith('*') or
                candidate.startswith('x') or
                candidate.startswith('=') or
                candidate.startswith('Resource changes:')
            ):
                differences.append(candidate)

    return differences

if __name__ == "__main__":
    working_directory = sys.argv[1]
    file1 = os.path.join(working_directory, 'base.txt')
    file2 = os.path.join(working_directory, 'new.txt')

    differences = compare_files(file1, file2)

    if differences:
        for diff in differences:
            print(diff)
    else:
        print("no changes")
```

## Operational workflow

When the pipeline fails:

1. Review `what-if` differences (“before/after” values).
2. Decide if the change was:

   * intended (CAB-approved, emergency fix, etc.), or
   * unintended drift that must be rolled back.
3. If intended, update the Bicep templates and baseline output so “desired state” stays accurate.
4. If unintended, revert the environment (manually or via controlled deployment), then re-run the baseline.

## Challenges / trade-offs

### ARM → Bicep conversion isn’t perfect

VS Code conversion works well, but exported templates often create:

* warnings / “yellow squiggles”
* properties that don’t round-trip cleanly
* noisy diffs from ordering or formatting

This is manageable but time-consuming during initial setup.

### Baseline maintenance

Production shouldn’t change frequently, but when it does, the baseline must be updated.
The benefit is that the system stays honest: **every intentional change becomes part of the documented desired state.**

## Where this fits well

* environments with strict change control (CAB / ITIL)
* teams that want drift visibility without building a full configuration management platform
* audit readiness: “we detect and review configuration drift” is an easy story to defend

## Notes / future improvements

Possible refinements depending on maturity and needs:

* compare structured JSON objects instead of text (reduce formatting noise)
* store baseline per resource type / module for more precise scope
* enrich alerts (e.g., email/Teams + links to diff artifact)
* integrate with policy-as-code for additional control signals

```


