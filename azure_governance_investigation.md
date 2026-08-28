# Investigating an Unauthorized Azure Deployment and Governance Failure

> Lab-specific resource names, tag values, deployment details, timestamps, and flags have been withheld to preserve the integrity of the exercise.

## Scenario

A junior intern received temporary Contributor access to deploy a test environment for an internal experiment. The intern completed the deployment late Friday afternoon without validating it against the organization’s governance standards. On Monday morning, I investigated the environment to determine what was created, trace the deployment, and identify why Azure Policy did not prevent the violation.

## Environment

* Live, multi-user Azure training tenant
* Reader-level investigator access

## Investigation

### 1. Identified a naming-convention violation

I began by reviewing the resource groups within the subscription and comparing their names against the organization’s documented naming convention.

Most resource groups followed a consistent structure containing a resource-group prefix, workload identifier, environment designation, region, and instance number. One resource group did not follow this pattern, making it a clear governance outlier requiring further investigation.

The exact resource-group name has been withheld because it is a lab answer.

### 2. Inspected the deployed resource and its metadata

I opened the non-compliant resource group and reviewed its contents. The group contained a single resource associated with the intern’s test environment.

I then inspected the resource’s Tags blade. An intern-specific tag connected the resource to the temporary deployment and provided useful ownership context. Although the intern applied the expected tag, the surrounding resource configuration still violated the organization’s naming standard.

I withheld the tag value from this write-up to preserve the lab exercise.

<img width="1889" height="878" alt="tags" src="https://github.com/user-attachments/assets/8b31c63e-5a53-45fc-87d7-0a3df5ea9f45" />

### 3. Traced the deployment

I reviewed the resource group’s Deployments blade to determine how the resource was provisioned.

The Azure Resource Manager deployment record showed the deployment’s status, configuration inputs, and creation timeline. This established a traceable provisioning event and connected the resource to the intern’s deployment activity.

I withheld the deployment name and timestamp because they are answers to the lab.

<img width="1891" height="635" alt="deployed_resource" src="https://github.com/user-attachments/assets/59153f41-d9f6-41d5-95f8-d73d01f880ef" />

### 4. Reviewed policy compliance

I examined the resource group’s Policies blade and found that the naming-convention policy had marked the environment as non-compliant.

This demonstrated that the policy successfully detected the violation. However, detection alone did not explain why Azure allowed the deployment to succeed.

<img width="1883" height="647" alt="policy_compliance" src="https://github.com/user-attachments/assets/262dbbe4-9010-4352-94b5-9a17bbd51fa4" />

### 5. Identified the governance control failure

I opened the applicable policy assignment and reviewed its parameters. The naming policy’s effect was configured as `Audit`.

<img width="1887" height="598" alt="naming_convention" src="https://github.com/user-attachments/assets/866aedef-71cb-46b6-9578-b25fa02daa8d" />

An `Audit` effect records a compliance violation but does not block the underlying request. A `Deny` effect would have rejected the non-compliant deployment.

I concluded the policy operated as configured: it detected the violation but allowed the deployment because it was not configured as a preventive control.

## Findings

1. A resource group was created outside the organization’s approved naming convention.
2. The resource group contained a resource associated with the intern’s temporary test deployment.
3. Resource tags provided ownership context but did not compensate for the naming violation.
4. Azure Resource Manager retained a traceable deployment record.
5. Azure Policy detected and reported the non-compliant environment.
6. The policy assignment used the `Audit` effect, allowing the deployment to proceed.
7. Temporary Contributor access provided the intern with sufficient permissions to create the environment.
8. No evidence reviewed during this investigation demonstrated malicious activity or identity compromise.

## Root Cause

The primary control failure was a mismatch between the intended governance requirement and the policy’s configured enforcement behavior.

The organization expected the naming policy to prevent non-compliant deployments, but the policy was configured to audit violations rather than deny them. Temporary Contributor access increased the importance of having correctly configured preventive controls.

## Recommended Actions

* Determine whether the naming policy was intentionally placed in audit mode.
* Test the policy against existing workloads before changing its effect to `Deny`.
* Document approved exemptions for resources that cannot follow the standard.
* Limit temporary users to the minimum permissions and scope required.
* Remove temporary role assignments when the authorized work is complete.
* Require deployments through approved templates or pipelines.
* Enforce ownership, environment, cost-center, and expiration tags.
* Establish an expiration and cleanup process for temporary environments.
* Review high-impact changes completed outside approved change windows.

## Evidence Collected

The investigation included screenshots of:

1. The subscription’s resource-group inventory.
2. The non-compliant resource group’s Overview blade.
3. The resource contained within the group.
4. The resource’s Tags blade.
5. The resource group’s deployment history.
6. The successful ARM deployment details.
7. The policy compliance result.
8. The policy assignment showing the `Audit` effect.

Lab answers have been blurred or cropped from all published screenshots.

## What broke / what surprised me. 

An active policy does not represent a preventive control; the naming policy existed and detected the violation; however, I still needed to inspect the parameters to understand why the deployment succeeded.

## What I learned. 

Naming conventions help investigators identify anomalous resources at scale, and legitimate contributor access can still introduce risk when preventive governance controls are incomplete. Lastly, don't ship test environments at the end of the day on Friday.  

## Conclusion

The investigation determined that the intern deployed a test resource using temporary Contributor access. Azure Policy detected the resulting naming violation but did not prevent it because the applicable assignment was configured with the `Audit` effect rather than `Deny`.

This investigation demonstrates the difference between detective and preventive cloud controls. Effective Azure governance requires properly scoped access, consistent resource standards, traceable deployments, and policies configured to enforce—not merely report—the organization’s requirements.
