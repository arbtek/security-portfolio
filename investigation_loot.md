# The Unprotected Copy
## Tracing Azure storage, identity, private networking, and secrets

**Siavash Sean Etesham · Mad Hat Chapter 6 · Azure security lab**

### Investigation summary

A developer wanted to know what their application reached through a connection string stored in Azure Key Vault, how the data was reachable, and whether it was exposed to the internet.

I traced the primary storage account through its authentication settings, managed identity permissions, private endpoint, and Key Vault secret. The configuration supported the intended private, identity-based access pattern. The more significant finding came from reviewing the rest of the resource group: a separate storage account contained a customer export that could be listed and read without authentication.

The primary account's controls still protected that account. They did not protect the export stored elsewhere.

![Primary storage protected by identity and private networking, beside a second copy reachable anonymously](assets/storage-contrast.svg)

### Scope and authority

This was an authorized training environment with lab data. My access consisted of **Reader** on the resource group and **Key Vault Secrets User** on the designated vault. The exercise permitted inspection and secret retrieval, with no configuration changes.

I reviewed the resource inventory, storage properties, role assignments, private endpoint configuration, and the designated secret. The lab also authorized an unauthenticated browser check against the second account. Remediation and detection below are recommendations; I did not deploy them.

## 1. The data: establish the resource behind the application

I started with the primary storage account and identified its account kind, redundancy, and blob service endpoint.

| Item | Observation | Interpretation |
| --- | --- | --- |
| Account kind | StorageV2 | General-purpose v2 storage account |
| Redundancy | Geo-zone-redundant storage (GZRS) | A resilience configuration, separate from access control |
| Blob endpoint | Recorded and correlated with the lab's data-access trace | Identifies the storage service being investigated |

The endpoint gave me a resource to follow across configuration, networking, and identity. The fact that Azure publishes a service hostname does not, by itself, establish that the data is publicly readable.

GZRS answered a different question: how the data is replicated for resilience. Redundancy does not decide who can retrieve a blob, and it does not compensate for an exposed export.

## 2. The key posture: determine how access is authorized

The primary account had **Allow storage account key access disabled**. I then checked the application's user-assigned managed identity in IAM and found **Storage Blob Data Contributor** assigned at the storage account scope.

This is a blob-data role, not the general Azure **Contributor** management role. It grants read, write, and delete access to blob data. The application therefore had more than read-only data permissions. Whether delete capability and account-wide scope were necessary would require a review of the application's actual operations. [1]

Disabling Shared Key blocks account-key authorization and account-key-signed SAS requests for Blob Storage. It does **not** rule out a user delegation SAS backed by Microsoft Entra ID, and it is not a substitute for checking anonymous access separately. [2]

The useful conclusion was specific: the primary account rejected the account-key path, and the application's identity had an explicit blob-data role. I did not treat one toggle as proof that every possible access path was secure.

## 3. The network: confirm the intended private route

The primary account's public network access was disabled. An approved private endpoint was present, and its DNS configuration showed a private IP address in the endpoint subnet.

Those observations supported the intended path: an application with suitable network connectivity resolves the storage hostname to the private endpoint and reaches Blob Storage over Private Link. Public network access must be checked separately because creating a private endpoint does not automatically disable the public endpoint. [3]

I inspected configuration rather than capturing traffic from the running application. I did not record a DNS lookup from the application host, validate its VNet integration, or execute a storage request from that host. Those would be the next checks before calling this an end-to-end runtime validation.

Likewise, private access is not limited to a client physically inside one VNet. A connected network can use the endpoint if routing and DNS are configured appropriately. The endpoint configuration establishes the intended route; it does not prove every client's ability to use it.

## 4. The secret: separate vault access from storage access

My resource-group Reader role allowed me to inspect configuration. The separate **Key Vault Secrets User** assignment allowed me to retrieve the designated secret's current value through the vault's data plane.

The application pattern described by the lab uses a managed identity to authenticate to Key Vault and retrieve the value at runtime. Microsoft documents this approach for applications that need secrets without embedding credentials for vault access in their code. [4]

Vault access and storage access are separate authorization decisions. Permission to retrieve a secret does not automatically grant permission to read blobs, and a connection string alone does not establish network reachability.

I completed the authorized secret retrieval, but did not review application source code, deployment settings, or runtime secret-access logs. I therefore would not claim to have proved that no credentials existed elsewhere or that every application request used the intended identity. Those conclusions need additional evidence.

### Response to the developer

> The primary data account has Shared Key access disabled and public network access disabled. Its approved private endpoint and DNS configuration support the intended private route, and the application identity has Storage Blob Data Contributor permissions. I also retrieved the designated Key Vault secret under the lab's scoped authorization. Validating DNS and a data request from the application host would complete the runtime check. Separately, I found a second storage account exposing a customer export anonymously; that requires attention beyond the original ticket.

## 5. The second copy: widen the review

Before closing the ticket, I reviewed the resource-group inventory and investigated a second storage account. Its configuration differed from the primary account:

| Control or test | Primary account | Second account |
| --- | --- | --- |
| Public network access | Disabled | Enabled |
| Shared Key authorization | Disabled | Not established in this review |
| Private endpoint | Approved endpoint observed | Not needed for the observed anonymous path |
| Account allows anonymous blob access | Not separately documented in my notes | Allowed |
| Unauthenticated container listing | Not tested | Listing returned |
| Unauthenticated export read | Not tested | JSON export returned |

The second account's two account-level settings were warning signs, but they were not enough to prove that a particular blob was exposed. Azure also requires the container to permit anonymous access. A container can permit blob reads without permitting listing; container-level public access can permit both. [5]

The decisive evidence was the browser result. Following the lab's unauthenticated test procedure, I obtained a container listing and then opened the listed JSON export without supplying credentials or a SAS token. The listing exposed object metadata; the subsequent request returned the export content. That moved the finding from a potentially unsafe setting to confirmed anonymous read access in the lab.

The lab scenario explained that a temporary export had been created for migration testing and left behind afterward. That history came from the exercise narrative. My observations independently established the exposure, but did not establish who created the account, when the exposure began, or whether another party had accessed it.

## Finding: anonymous access to a customer-data export

**Assessment:** High priority for remediation in a comparable production environment. This is a qualitative assessment of the scenario, not a scored production incident.

**Affected resource:** The separate export account and its publicly accessible container.

**Impact demonstrated:** An unauthenticated client could discover the export through listing and retrieve its contents. The primary account's identity, network, and secret controls were outside that request path.

**Not demonstrated:** Anonymous write or delete access, access to every container in the account, a byte-for-byte comparison with the primary dataset, or access by a third party.

The failure was a gap in the protection of data copies. Reviewing only the primary resource would have produced a reassuring answer while leaving the exposed export unaddressed.

## Recommended response and verification

These are proposed production actions for the resource owner and incident-response team, not changes performed in this read-only lab.

1. **Record the exposure and identify dependencies.** Preserve relevant configuration, timestamps, and available logs while coordinating prompt containment with the owner. Establish whether any legitimate consumer relies on the public path.
2. **Remove anonymous access.** Set the account's `allowBlobPublicAccess` to `false` and make the affected container private. If public network access is unnecessary, disable it after confirming the replacement private route. [6]
3. **Retest both sides.** From an unauthenticated external client, confirm that listing and a known export URL no longer return data. From the approved application path, verify that required operations still succeed.
4. **Determine the exposure window.** Review resource changes and available blob access logs. An absence of historical logs limits the ability to determine who accessed the data; it does not establish that nobody did.
5. **Review the export's contents and disposition.** If it contains reusable credentials, coordinate their rotation or revocation. After retention and evidence requirements are addressed, remove the unnecessary copy or move it to approved storage.
6. **Find other copies.** Review export, migration, test, and backup storage beyond this resource group using the appropriate authorized scope. Give temporary resources an owner and an expiry date.

## Detection and prevention at scale

I would combine configuration enforcement with data-access monitoring rather than expect one alert to cover both.

| Layer | Proposed control | What it establishes |
| --- | --- | --- |
| Prevention | Assign an appropriate Azure Policy with a Deny effect against anonymous blob access where supported; review documented exceptions | Blocks disallowed account configurations before deployment succeeds |
| Posture | Inventory storage accounts that allow anonymous access; review container settings and network exposure | Finds exposure candidates, including accounts that receive no traffic |
| Change monitoring | Monitor successful account and container changes, then evaluate resulting configuration | Detects regressions; a generic write event alone does not prove public access |
| Data-access monitoring | Collect Blob read logs and alert on successful anonymous listing or retrieval for accounts intended to be private | Shows use of the anonymous access path |

The included `detections.kql` contains an Azure Resource Graph posture query and a Log Analytics query for successful anonymous reads. They are proposed starting points, checked against the documented schema but **not executed in this lab**. The access query requires resource-specific Blob logs in `StorageBlobLogs`; log collection is not retroactive. [6][7]

For a newly created storage account, Policy provides the preventative check. Inventory and log alerts add coverage for existing resources and changes. An alert after access has occurred cannot replace a deployment control.

### Acceptance criteria for the recommendations

- Anonymous listing and a direct request for the known export return no data after containment.
- Approved application reads and writes still work through the intended identity and private route.
- A controlled attempt to deploy a disallowed account configuration is denied by the assigned policy.
- An authorized test in an isolated validation environment produces the expected telemetry and alert before that environment is removed.
- Every reviewed export has an accountable owner, retention requirement, and approved access model.

## Evidence and confidence

| Evidence in my working notes | Supports | Limit |
| --- | --- | --- |
| Storage overview / JSON and Configuration views | Account kind, redundancy, endpoint, and key posture | Point-in-time configuration |
| Managed identity IAM view | Storage Blob Data Contributor assignment and account scope | Does not prove which operations the application needs or performs |
| Networking and private endpoint DNS views | Disabled public network setting and configured private endpoint | No application-host connectivity capture |
| Authorized Key Vault retrieval | Ability to read the designated secret under the scoped role | No application-code or runtime-log review |
| Second-account JSON, anonymous listing, and export response | Public-access allowance and successful anonymous retrieval | No historical access analysis or full dataset comparison |

## What I learned

The first four stops established what the intended protected path looked like. The fifth changed the scope of the answer.

I needed to distinguish a role from its scope, a published hostname from public data access, and a configured private endpoint from a tested application connection. Most of all, I needed to ask where else the data lived.

Protecting the primary account is necessary. Protecting customer data also means finding and securing its exports, backups, and temporary copies.

## References

The Mad Hat Chapter 6 lab and portfolio guide supplied the scenario. The observations above come from my lab notes. These Microsoft references support the Azure control explanations and proposed follow-up work:

1. [Azure built-in roles for Storage: Storage Blob Data Contributor](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles/storage#storage-blob-data-contributor)
2. [Prevent Shared Key authorization](https://learn.microsoft.com/en-us/azure/storage/common/shared-key-authorization-prevent)
3. [Use private endpoints for Azure Storage](https://learn.microsoft.com/en-us/azure/storage/common/storage-private-endpoints)
4. [Use a managed identity to connect Key Vault to an Azure web app](https://learn.microsoft.com/en-us/azure/key-vault/general/tutorial-net-create-vault-azure-web-app)
5. [Configure anonymous read access for containers and blobs](https://learn.microsoft.com/en-us/azure/storage/blobs/anonymous-read-access-configure)
6. [Remediate anonymous read access to blob data](https://learn.microsoft.com/en-us/azure/storage/blobs/anonymous-read-access-prevent)
7. [Azure Monitor: StorageBlobLogs schema](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/storagebloblogs)
