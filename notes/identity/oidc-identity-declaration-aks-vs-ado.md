# OIDC Identity in AKS vs Azure DevOps: Same Protocol, Different Declaration Point

Both AKS Workload Identity and Azure DevOps Pipelines use the same underlying mechanism:
**OIDC token exchange with Entra ID**.

What differs — and matters operationally — is **where the identity is declared**.

That single difference changes:
- how configuration is expressed
- who “owns” identity decisions
- how least privilege is enforced
- how reviewers reason about blast radius

---

## Same protocol, different identity declaration point

### AKS (Workload Identity): the workload declares itself

In AKS, identity is **explicitly declared by the workload at runtime**.

The Kubernetes primitives *are* the identity boundary:

- **Namespace**
- **ServiceAccount**

A pod runs *as* a specific ServiceAccount in a specific namespace, and the OIDC token minted for that pod carries this identity directly in its claims.

From Azure’s perspective, identity is derived from Kubernetes context:

- **Issuer**: the AKS cluster’s OIDC issuer
- **Subject (`sub`)**:  
  `system:serviceaccount:<namespace>:<serviceaccount>`
- **Audience (`aud`)**: the configured audience for token exchange

The pod is effectively saying:
> “I am this service account, in this namespace.”

#### Configuration impact

Identity configuration is split across **Kubernetes + Entra**:

- **Kubernetes**
  - create or select namespace
  - create ServiceAccount
  - bind RBAC if needed
  - set `serviceAccountName` on the pod
- **Entra ID**
  - create a federated credential that matches *exactly*:
    - issuer
    - subject
    - (optionally) audience
- **Azure RBAC**
  - grant permissions to the identity used by that federated credential

**Net effect:**  
Identity is *close to the workload*.  
Anyone reviewing the manifests can see **who this pod runs as**.

---

### Azure DevOps Pipelines (OIDC): the platform declares identity

In Azure DevOps, the pipeline YAML does **not** meaningfully declare identity to Azure.

Instead:

- the pipeline selects a **service connection**
- the **Azure DevOps control plane** mints the OIDC token
- identity claims are derived from ADO’s own context:
  - organization
  - project
  - pipeline
  - service connection

From Azure’s perspective:

- **Issuer**: Azure DevOps’ OIDC issuer
- **Subject (`sub`)**: a value defined by ADO to represent the pipeline/service-connection context
- **Audience (`aud`)**: expected by Azure for token exchange

The pipeline is effectively saying:
> “Use *that* service connection.”

ADO supplies the identity claims.

#### Configuration impact

Identity configuration lives primarily in **ADO + Entra**, not YAML:

- **Azure DevOps**
  - create Service Connection (Workload Identity Federation)
  - control which pipelines can use it
  - apply approvals / environments / permissions as governance
- **Entra ID**
  - create federated credential matching ADO’s issuer + subject
- **Azure RBAC**
  - grant permissions to the identity

**Net effect:**  
Identity is *owned by the platform*.  
Reviewers must reason about identity via **ADO settings**, not just pipeline YAML.

---

## The clean distinction

> **AKS:** The workload’s identity is declared in Kubernetes (namespace + serviceAccount), and Azure validates that exact `sub` from the cluster issuer.  
> **ADO:** The workload’s identity is declared by Azure DevOps (service connection + pipeline context), and Azure validates whatever `sub` ADO asserts — YAML mostly selects the service connection, it doesn’t define identity claims.

---

## Why this matters in practice

### In AKS, identity lives “in the pod’s world”

- You can point at a manifest and say:  
  “This pod runs as **ServiceAccount X in namespace Y**.”
- The identity boundary is **namespace + ServiceAccount**.
- Least privilege naturally aligns to workload boundaries.

**Common AKS identity knobs:**
- `serviceAccountName`
- namespace placement
- ServiceAccount annotations (WI enablement)
- Kubernetes RBAC
- Entra federated credential matching issuer + subject

---

### In ADO, identity lives “in the DevOps world”

- Pipeline YAML usually shows only:  
  “Uses **Service Connection Z**.”
- The identity boundary is **who can use that service connection**.
- Least privilege is enforced through **ADO governance**, not YAML structure.

**Common ADO identity knobs:**
- Service Connection definition (WIF)
- service connection permissions
- pipeline permissions / approvals / environments
- Entra federated credential matching ADO issuer + subject

---

## Minimal examples

### AKS: identity is explicit in the workload spec

```yaml
spec:
  serviceAccountName: saas-node-creator-sa
````

…and namespace is part of the identity boundary:

```yaml
metadata:
  namespace: saas
```

---

### ADO: identity is implied by selecting a service connection

```yaml
steps:
- task: AzureCLI@2
  inputs:
    azureSubscription: "SC-Prod-SaasNodeCreator"
```

That YAML does not declare identity to Azure — it tells **ADO** which identity plumbing to use, and ADO supplies the claims.

---

## Security posture takeaway

* **AKS:** least-privilege is enforced by *workload boundaries*.
  Identity is small by construction.
* **ADO:** least-privilege is enforced by *governance boundaries*.
  Identity tightness depends on service connection discipline and permissions.

Both are secure when used correctly — but they demand **different mental models**.

```

