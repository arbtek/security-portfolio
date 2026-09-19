# The Privilege Audit

## Azure RBAC & PIM audit report

**Siavash Sean Etesham · September 2026**  
[Website case study](https://siavashetesham.com/projects/privilege-audit)

## Executive summary

I completed a five-stage privilege-access audit in the Mad Hat Labs Azure training tenant. Using four complementary methods, I investigated repeated Owner grants, validated a Reader assignment left behind after account deletion, compared active access with eligibility, and inspected an overprovisioned Owner assignment in a resource group outside my initial view.

The central result was a repeatable audit methodology: each method answered a different access question, and each required additional evidence to close its blind spots. The final Owner record also contained a condition, reinforcing that role name, scope, and delegation limits must be evaluated together.

Findings and proposed corrective actions are documented below.

## Scope and methodology

| Scope item | Audit boundary |
| --- | --- |
| Environment | Mad Hat Labs live Azure training tenant; identifiers withheld |
| Access | Restricted Reader access, followed by the lab's authorized PIM activation path; the final resource view showed a time-bound Operative assignment |
| In scope | Azure resource role assignments, principal references, scope, inheritance, eligibility, active state, and assignment conditions |
| Outside this audit | A comprehensive Entra directory-role review, complete group-membership review, tenant-wide assurance, and production remediation |
| Tools and evidence | IAM blade / CSV export; Azure CLI and supplied JSON comparison; Resource Graph / KQL; PIM assignment export and access-state views |
| Completion | Five stages completed using four tools; the fifth stage combined their results |

Utilized a role assignments CSV from the IAM blade export. I also exported a roles .json file for the investigation.

### Methodology comparison: what each method reveals and misses

| Method | Reveals / best use | Limits and blind spots | How I addressed the gap |
| --- | --- | --- | --- |
| IAM blade / role-assignment export | Baseline of principals, roles, scope, and inherited grants visible in the selected view/export | Group assignments do not expand into a complete membership review. Unresolved identities require deliberate inspection; the portal can display “Identity not found.” Export options and permissions affect coverage. | Compared the baseline with CLI/JSON evidence and tracked the principal ID privately. |
| Azure CLI | Scriptable enumeration with principal IDs, roles, scopes, and unresolved-name fields | My lab command targeted one resource group. A broader scope is possible with CLI options and subscription iteration. Empty names can result from failed directory lookups or deletion; ordinary assignment listing does not inventory all PIM eligibility. | Compared live output with the supplied name-resolved export before classifying the orphan. |
| Resource Graph / KQL | Cross-scope correlation across selected, accessible subscriptions; efficient principal-ID searches | The roleAssignments query used here covers assignment records, not a full PIM eligibility inventory. Results remain permission-bound; a directory selection does not guarantee complete tenant coverage. | Recorded the query boundary and added a separate PIM review. |
| PIM assignment export and audit views | Eligible versus active state, assignment duration, and direct/inherited context; separate audit views can provide activation history | An assignment export is not an activation-history export. Resource selection and permissions constrain coverage; one export does not prove every standing grant or group access path has been reviewed. | Reconciled PIM with ordinary RBAC evidence and inspected effective access at the final resource group. |

These are the limits of the specific views and queries used, rather than absolute limits of each product. Azure CLI supports broader enumeration, and the current IAM portal can expose eligible/time-bound assignment state. PIM views can also show permanent active assignments. [Azure CLI options](https://learn.microsoft.com/en-us/cli/azure/role/assignment#az-role-assignment-list) · [Azure RBAC/PIM integration](https://learn.microsoft.com/en-us/azure/role-based-access-control/pim-integration)

The fifth stage was synthesis: use the authorized activation path, inspect the additional resource scope, and evaluate the final assignment in context. I did not treat it as a fifth independent data source.

## Severity-ranked findings

Severity is an analyst assessment of the lab conditions, not a CVSS score. High means broad management privilege requiring priority review; Medium means a narrower stale entitlement and lifecycle-control gap. Validate business need and production impact with the resource owner before changes.

| ID | Priority | Finding class and evidence | Risk and recommended disposition |
| --- | --- | --- | --- |
| A-01 | High | Redundant Owner grants for one user across multiple scopes; IAM baseline, Figure 1 | Broad privilege and overlapping grants complicate revocation. Validate the full scope map and replace unnecessary Owner grants with the narrowest job-function role and scope. Scripted creation is a hypothesis, not a confirmed root cause. |
| A-02 | High | Overprovisioned Owner assignment at the final resource group, with a version 2.0 condition; Figure 6 | The condition limits relevant delegation operations but does not establish least-privilege resource management. Review the role, condition, and business requirement together. |
| A-03 | Medium | Orphaned Reader assignment tied to the principal identified as deleted in the lab; supplied JSON and scenario context, Figure 2 | Identity deletion and permission cleanup were not reconciled. Confirm lifecycle status, remove the exact obsolete assignment, and check for remaining grants. |

**Additional governance review:** Inventory permanent active privileged assignments and determine which human administrative tasks require standing access. Move suitable intermittent tasks to eligible access. The screenshots do not establish the business necessity of every permanent grant or prove that MFA, approval, or justification controls were disabled; I do not report those as confirmed defects.

## Audit procedure and supporting evidence

### Stage 1 — IAM baseline and redundant Owner grants

I began with the IAM CSV. One user appeared repeatedly with Owner assignments, including a subscription-level grant identified by the lab and further grants across resource scopes.

The repetition made the account a priority for least-privilege review. Repeated grants are consistent with bulk provisioning, but the evidence does not establish whether a script or a person created them. The supplied export was the working evidence; the screenshot's truncated scope column does not support an exact count of distinct affected resources.

The cleanup question is more subtle than removing a few duplicate rows. An explicit child-scope assignment can remain after a parent assignment is removed. Conversely, removing child assignments may leave the account's broader inherited access unchanged. I would review those grants together against the account's actual responsibilities.

<img width="1108" height="736" alt="01-iam-baseline" src="https://github.com/user-attachments/assets/2333c4a6-d10c-4b14-879a-34cad6a3cfd9" />

*Figure 1. Repeated Owner assignments in the supplied baseline. USER-A is an added anonymized label for the same account; names, scope paths, IDs, and descriptions are removed.*

### Stage 2 — CLI enumeration and orphan validation

I ran a resource-group query in Azure CLI from a PowerShell session in Cloud Shell, then compared the output with the supplied JSON export. The published command replaces the actual resource-group name:

```powershell
az role assignment list --resource-group "<RESOURCE-GROUP>"
```

The relevant JSON record retained a principal ID and resource-group scope. Its `principalName` was empty, its `principalType` was `User`, and its assigned role was **Reader**. The lab explanation identified it as an account deleted without removing the assignment.

The comparison mattered because my restricted account could also encounter missing names during directory lookups. A blank name in my own command output was not enough to classify an identity as deleted. The supplied export and scenario context distinguished the intended orphaned record from that visibility limitation.

This was an assignment-lifecycle finding. The record did not establish that the deleted account could still authenticate or that an attacker had used it. In an operational audit, I would confirm the object's lifecycle status before approving removal. Microsoft also documents replication delays as a possible cause of unresolved identity names. [Reference](https://learn.microsoft.com/en-us/azure/role-based-access-control/troubleshooting#symptom---role-assignments-with-identity-not-found)

#### Why the orphan matters

An Azure role assignment references a principal's object ID, not its display name. Deleting the identity does not automatically remove the assignment. Microsoft states that a remaining assignment can continue to authorize a deleted principal while valid, unexpired Entra tokens exist. That is a concrete reason to treat stale assignments as revocation work rather than cosmetic cleanup. [Microsoft: role assignments and deleted principals](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments#role-assignments-with-identity-not-found)

The lab did not demonstrate surviving tokens or exploitation. The evidence also does not show that an attacker could create a new account with an arbitrarily chosen object ID. Reusing a display name does not establish identity continuity. If an original identity is restored, review its associated access as part of restoration rather than assuming it is safe.

<img width="903" height="289" alt="02-orphaned-reader" src="https://github.com/user-attachments/assets/fbc0aeb5-6173-4f9d-b892-e38c0a9f9628" />

*Figure 2. The supplied export retained a Reader assignment for the lab's deleted account. The empty name, principal type, role, and resource type remain visible; identifiers, timestamps, paths, and the challenge answer are removed.*

### Stage 3 — Resource Graph scope correlation

I moved to Azure Resource Graph Explorer and queried `authorizationresources`. The preserved screenshot shows an inventory query restricted to role-assignment records, projecting principal ID, principal type, scope, and description. It returned **83 records** in the accessible query scope.

That screenshot has no principal-ID filter. The 83 records are therefore the broad inventory result—not 83 orphaned assignments. For the targeted lab stage, I carried forward the orphaned principal ID; the lab review records the accepted identifier. The targeted result count was not preserved in this screenshot.

The corrected query pattern for that pivot is:

```kusto
authorizationresources
| where type =~ 'microsoft.authorization/roleassignments'
| extend principalId = tostring(properties.principalId)
| extend description = properties.description
| where principalId == '<REDACTED-PRINCIPAL-ID>'
| project name, principalId,
          principalType = properties.principalType,
          scope = properties.scope, description
```

Resource Graph can query across multiple accessible subscriptions. It does not bypass RBAC, and a directory-level selection does not grant tenant-wide visibility. I treated the results as evidence of the scopes the account could inspect, not proof that inaccessible areas were clear. [Reference](https://learn.microsoft.com/en-us/azure/governance/resource-graph/overview)

<img width="1900" height="878" alt="03-resource-graph" src="https://github.com/user-attachments/assets/586948d4-059f-4c31-9ab4-93597a342dfc" />

*Figure 3. The broad role-assignment query and its actual result count. Principal IDs, assignment IDs, tenant details, and exact scope values are redacted. The query shown is unfiltered by principal.*

### Stage 4 — PIM eligibility, active access, and duration

The earlier stages focused on ordinary active role assignments. PIM added the distinction between access that was already active and access available for activation.

I reviewed the resource-group assignment export, which included assignment state, direct or inherited membership, duration, and resolved-name context. The account needed for the fourth-stage clue appeared with a direct, permanent **Reader** assignment. Its name was the clue; the role record defined its permission. Nothing in that excerpt demonstrated disabled MFA, absent approval, or a specific activation-policy defect.

The later PIM capture showed an eligible **Member** assignment under **My roles → Groups**. That is group-membership eligibility, a distinction worth preserving when explaining how the audit account obtained access. The capture alone does not enumerate the group's complete resource entitlements.

<img width="926" height="381" alt="04-pim-eligibility" src="https://github.com/user-attachments/assets/bd152876-3ffc-4d53-ac67-0377b113604b" />

*Figure 4. Eligible group membership in PIM. Permanent here describes the eligibility shown in this view; it does not establish a permanently active resource privilege. The group name is removed.*

For Azure resource roles, the PowerShell commands covered in the exercise query eligibility and assignment schedules separately:

```powershell
Get-AzRoleEligibilitySchedule -Scope "<AZURE-RESOURCE-SCOPE>"
Get-AzRoleAssignmentSchedule -Scope "<AZURE-RESOURCE-SCOPE>"
```

These are documented query examples, not additional captured command executions. Eligibility requires activation before use; active access can be permanent or time-bound. Just-in-time access limits when a privilege is available. The role and scope must still be appropriate. [Reference](https://learn.microsoft.com/en-us/azure/role-based-access-control/pim-integration)

### Stage 5 — Synthesis at the final resource group

I used the lab's authorized activation path to continue into the additional resource group. The resulting **Check access** view showed two active assignments and one eligible assignment for my audit account. The expanded rows included an active, time-bound custom **Operative** assignment at “This resource,” alongside an eligible Operative assignment and an inherited permanent assignment.

My audit account held temporary Operative access. The Owner assignment examined below belonged to the separate finding under investigation.

<img width="936" height="863" alt="05-effective-access" src="https://github.com/user-attachments/assets/9e20b3ea-d9a1-456a-844d-55d6c576eb80" />

*Figure 5. The audit account's access states at the final resource group. The active time-bound Operative row is distinct from the separate Owner assignment under investigation. Account, group, and resource names are removed.*

The accompanying resource-group export contained the **Owner** assignment used to complete the hunt. It also carried a **version 2.0 condition** referencing role-assignment write and delete operations and a set of privileged role definitions.

That condition is material. An Owner label should not automatically be described as unrestricted access delegation when the assignment contains a relevant condition. The condition shown limits the matching role-assignment operations; it is not evidence that all resource-management authority has been reduced to a narrow operational role. Evaluate the role, scope, and condition together. [Reference](https://learn.microsoft.com/en-us/azure/role-based-access-control/delegate-role-assignments-portal)

<img width="1217" height="207" alt="06-owner-evidence" src="https://github.com/user-attachments/assets/0227275d-728b-4477-8962-83b4bb061b76" />

*Figure 6. The final export's Owner row and condition. Lab identifiers and the answer are removed. The remaining GUIDs are Microsoft's public built-in role-definition identifiers, preserved to keep the condition interpretable.*

The exercise was complete when we recovered the final finding. The outcome was an access audit, with the Owner finding documented in its actual context.

## Recommendations and closure criteria

The following actions are proposed. Each finding should become an owned remediation ticket, with its assignment reference retained privately, a due date, evidence, and a closure decision.

| Action | Proposed owner | Closure evidence |
| --- | --- | --- |
| Revoke confirmed orphaned assignments | IAM administrator | Deleted-principal status verified; the exact assignment removed; fresh queries show no remaining obsolete grants in reviewed scopes |
| Reduce redundant Owner grants | Resource owner and IAM administrator | Approved task-to-role mapping; narrowest suitable scope; parent and child grants reconciled; required operations succeed while unnecessary capabilities fail in an authorized test |
| Replace unnecessary standing human privilege with eligible access | IAM / PIM administrator | Appropriate eligible role or group membership; MFA or authentication-context requirements, justification, and activation time limits verified; approval where warranted; exceptions documented |
| Prefer governed group assignments where appropriate | Identity governance owner | Membership and group ownership reviewed; nested/dynamic membership implications considered; unnecessary direct grants removed after access validation |
| Review assignment conditions and equivalent access paths | Cloud security reviewer | Conditions assessed alongside direct grants, inheritance, group membership, and eligibility; no unreviewed alternative grants the same excessive authority |
| Run quarterly reviews and track findings as a ticket queue | Security governance owner | Defined scope, retained baseline, named reviewers, findings with owners and due dates, and evidence-based closure; additional reviews after significant access or lifecycle changes |

PIM eligibility recommendations apply to supported human-access scenarios. Workload identities require their own least-privilege and lifecycle controls; they should not be treated as people who can complete an activation prompt.

After changes, I would allow for propagation, refresh the access context, and rerun the relevant queries. A finding is closed when the intended access state is verified across the reviewed scope—not simply when one row disappears from one export.

## Evidence limits and publication controls

- The **83-record Resource Graph result is a broad inventory**, not a count of orphaned accounts or matched stale assignments.
- The PIM group screenshot establishes eligibility; the resource-group view establishes time-bound access.
- The Owner export includes a condition. Its restrictions remain part of the analysis.
- No activation-history review, complete group-membership audit, exact affected-resource total, or completed remediation is claimed beyond the captured evidence.
- Flags, challenge answers, account names, the orphaned principal ID, and **all role-assignment description values** are omitted. Exact resource paths, lab identifiers, and timestamps are also removed.
- USER-A is an added anonymized label. Remaining GUIDs in Figure 6 are public Microsoft built-in role-definition IDs, retained solely to interpret the condition.

## Audit conclusion

The four methods formed a stronger review together than any individual export. IAM established the baseline; CLI made enumeration repeatable; Resource Graph connected records across accessible scopes; PIM added eligibility and duration. The final hunt required combining that evidence under an authorized access change.

The audit identified excessive and redundant privilege alongside an orphaned assignment, and produced a remediation plan with measurable closure criteria. The methodology is the reusable outcome: establish coverage, correlate the same principal across views, account for each tool's blind spots, and verify the remaining effective access.

**Skills demonstrated:** Azure RBAC, IAM governance, Azure CLI, CSV/JSON analysis, Resource Graph/KQL, PIM eligibility and active-state review, assignment-condition analysis, and remediation validation planning.

**Lab credit:** Mad Hat Labs, Chapter 3.5, “Privilege Audit,” and the Week Three Portfolio Guide. This report documents a training exercise. Screenshots are flattened, redacted copies, and no answer key is included.
