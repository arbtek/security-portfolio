# The Friday Deploy: Azure Pre-Production Security Review

**Author:** Siavash Sean Etesham  
**Exercise:** Mad Hat Labs · Chapter 4, Compute Fundamentals  
**Environment:** Live, multi-user Azure training tenant  
**Access:** Reader for the deployment under review  
**Review disposition:** Hold production promotion pending corrective work and validation

I reviewed a newly deployed notification workload against the training environment's established production baseline. The review covered region placement, workload identity, storage exposure, and inbound network access. My deliverable was a prioritized remediation plan, with demonstrated anonymous data access placed ahead of configuration improvements.

This is a pre-production review. The production baseline is a reference stack within the lab; this report does not describe a customer incident or claim that I changed a production environment. Resource names, exact locations, identifiers, endpoints, deployment timestamps, and challenge answers are withheld.

## The Call: What Gets Fixed Today?

> Stop anonymous access to the exposed file first, then verify that it no longer loads without authorization. Hold the release until the remaining deployment gaps are corrected and tested.

My ranking followed the lab's principle: **exposure beats hygiene; active beats potential.** I could demonstrate that a file was readable from an unauthenticated session. That made it the first containment task. The other findings also mattered, but the available evidence did not demonstrate equivalent data disclosure through them.

| Order | Finding | Why this order | Recommended timing |
| --- | --- | --- | --- |
| 1 | Anonymous access to stored data | A file was already readable without sign-in | Contain immediately; repeat the unauthenticated check |
| 2 | Unrestricted public application ingress | A scheduled workload had an unnecessary public entry point | Correct promptly in the same change cycle, before promotion |
| 3 | No managed identity attached | Workload authentication differed from the reference design | Resolve authentication and permission design before promotion |
| 4 | Region placement differed from the baseline | Cross-region dependencies can add latency and transfer cost | Correct placement in the next deployment, before promotion |

This order reflects the evidence and workload purpose. I would reassess it if testing showed an exploitable application endpoint, disclosed credentials, or a binding data-residency requirement. A public endpoint without a routing layer is still discoverable; the absence of application traffic is not a security control.

## Context and Scope

The scenario involved a notification service deployed late on a Friday and reviewed before promotion to production. Its intended job was scheduled processing rather than serving public requests.

I had Reader access to the review scope. I inspected configuration, compared it with the reference deployment, and performed the lab-authorized anonymous read check. Changes to access controls, identity, application connections, and region placement were recommendations for an authorized owner.

No written deployment standard was provided. I derived a provisional baseline from the working reference stack. A working configuration is useful evidence of the platform's intended pattern, but it still needs owner confirmation before becoming a formal standard.

## Method: Compare the Deployment, Then Test the Exposure

I used five questions to structure the review: where is the workload, how does it authenticate, where does its data sit, who can reach it, and what gets fixed first?

| Review area | Reference pattern | New deployment observation | Evidence used |
| --- | --- | --- | --- |
| Region | Compute located in the platform's standard region | Compute located in a different region | Function App overview comparison |
| Identity | User-assigned managed identity attached | System-assigned identity off; no user-assigned identity attached | Identity configuration comparison |
| Storage | Private container access pattern | One container permitted anonymous blob reads | Container access setting and a successful unauthenticated read |
| Ingress | Restrictive inbound configuration | Public access enabled without inbound restrictions | Networking and access-restriction comparison |
| Priority | Resolve demonstrated exposure before promotion | Four findings required a sequenced response | Evidence, workload purpose, and expected impact |

The distinction between configuration and behavior was most important for storage. A permissive setting suggested a problem; the anonymous read established that the exposure was effective.

## The Where: Is It in the Right Place?

**Observed.** The new Function App was running on an App Service plan in a different region from the reference Function App. The scenario placed the platform's dependent data in its standard region.

**Verified.** I compared the region fields in the application overview pages. I did not measure dependency latency or inspect a billing statement.

**Why it matters.** Separating compute from its data can add network delay and transfer charges. Azure's bandwidth guidance distinguishes free inbound transfer from chargeable outbound inter-region transfer; actual cost depends on the service, path, volume, and applicable pricing. Region mismatch identifies a cost and performance concern, not a measured invoice amount. [Microsoft bandwidth pricing](https://azure.microsoft.com/en-us/pricing/details/bandwidth/)

**Recommended action.** Confirm each dependency's location and any residency constraints with the platform owner. Redeploy into the approved region, then compare application behavior, latency, and transfer usage. Encode the approved location in deployment configuration.

## The Who: How Does It Authenticate?

**Observed.** The new Function App had no managed identity attached. The reference app used a user-assigned managed identity.

**Verified.** I checked both system-assigned and user-assigned identity views, then compared them with the reference app. These observations establish an identity configuration gap. They do not independently prove that a credential was stored in application settings or exposed in a file.

**Why it matters.** Workload identity can replace application-managed credentials for supported connections. The review still needs to establish how the app actually authenticates; an absent identity alone does not answer that question.

**Recommended action.** Have the owner inspect the application's connections and code, attach the approved user-assigned identity, and grant only required permissions. Update supported connections to use it, validate operation, then remove obsolete credentials. Rotate or revoke a credential if exposure is confirmed. Functions host storage and individual bindings can need different permissions, so one generic storage role should not be assumed sufficient. [Microsoft Functions connection guidance](https://learn.microsoft.com/en-us/azure/azure-functions/manage-connections)

## The Leak: Where Does Its Data Sit?

**Observed.** One container permitted anonymous blob access, unlike the private access pattern around it.

**Verified.** I copied the file URL from its properties and opened it in a separate browser session where I was signed in to no account. The file loaded without a sign-in prompt. The exercise described this as a direct blob URL without a token; a signed URL or other supplied authorization would not demonstrate anonymous access.

**Why it matters.** This was demonstrated data exposure. It was not proof of third-party access, malicious download, anonymous listing of the container, or public write access. Blob-level anonymous access permits reads of known blob URLs without necessarily allowing enumeration. Effective access also depends on the storage account's anonymous-access setting. [Microsoft anonymous-access guidance](https://learn.microsoft.com/en-us/azure/storage/blobs/anonymous-read-access-configure)

**Recommended action.** Make the affected container private immediately. Where no legitimate public-data use exists, disallow anonymous blob access at the storage-account level after checking dependencies. Retest the same unsigned URL from a clean session and confirm the data is unavailable. Review available access logs and the exposed content to assess possible impact; rotate secrets only if they were actually present or otherwise exposed.

## The Door: Who Can Reach It?

**Observed.** Public network access was enabled, no inbound access restrictions were configured, and unmatched traffic was allowed. The reference application had restrictive inbound rules.

**Verified.** I compared the applications' networking configuration. I did not demonstrate remote code execution, bypass application authentication, or capture a successful external application request.

**Why it matters.** The network policy admitted traffic to a workload whose stated purpose did not require public inbound requests. Reachability and authorization are different controls; this finding establishes unnecessary network exposure, not anonymous permission to invoke every function.

**Recommended action.** Apply the approved deny configuration and explicitly verify the unmatched-traffic action. App Service implicitly denies unmatched traffic once rules exist only when that action has not been explicitly configured. Also review the separate SCM/Kudu endpoint and required deployment paths. Test both blocked public requests and successful scheduled execution. [Microsoft access-restriction guidance](https://learn.microsoft.com/en-us/azure/app-service/overview-access-restrictions)

## Recommendations and Release Acceptance

These are proposed closure criteria, not completed remediation results.

| Work item | Suggested owner | Evidence required before closure |
| --- | --- | --- |
| Remove anonymous data access | Storage/platform owner | Access settings corrected; fresh unsigned request cannot retrieve the file; legitimate workload access still succeeds |
| Restrict application ingress | Platform owner | Main and SCM policies reviewed; public access rejected as intended; scheduled work and deployment process tested |
| Implement workload identity | Application and platform owners | Required operations succeed through approved identity; unnecessary operations fail; obsolete credentials removed after validation |
| Align region placement | Platform/application owners | Compute and dependencies match the approved placement design; behavior and transfer impact checked |
| Make review repeatable | Platform lead | Versioned baseline, named reviewer, automated configuration checks, exception process, and retained validation evidence |

A deployment checklist should ask the same five questions. Configuration checks can flag drift before release; behavioral tests are still needed to confirm that intended access works and unintended access fails.

## Outcome and What I Learned

I documented four configuration findings and made a priority call grounded in the strongest observed evidence. My review disposition was to hold promotion, contain the demonstrated exposure, and require validation of the remaining corrections.

The most useful lesson was how to move from a setting to a defensible conclusion. The anonymous read demonstrated exposure; it did not demonstrate a breach. The identity view showed a missing managed identity; it did not establish the full authentication path. Being precise about those boundaries makes the remediation plan more credible.

The second lesson was how to turn an undocumented working system into a review baseline without assuming every existing choice is correct. Comparing the deployments exposed the differences; workload purpose, effective behavior, and owner-confirmed requirements determined what those differences meant.

**Skills demonstrated:** Azure compute review, baseline comparison, workload identity assessment, storage exposure validation, inbound access review, risk prioritization, and release acceptance planning.

**Credit:** Mad Hat Labs, Chapter 4, “The Friday Deploy.” This public report describes my review method and conclusions without publishing challenge answers or environment identifiers.
