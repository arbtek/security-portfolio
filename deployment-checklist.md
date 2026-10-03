# Compute Deployment Review Checklist

This is a proposed control derived from the review, not an implemented pipeline. Apply it to the workload's actual requirements and approved exceptions.

## The Where

- [ ] Compute and dependency locations match the approved architecture.
- [ ] Cross-region dependencies have a documented reason and cost/performance review.
- [ ] Residency requirements and region exceptions have an accountable owner.

## The Who

- [ ] The authentication path is documented for every dependency.
- [ ] Supported connections use the approved workload identity.
- [ ] Permissions match required operations and the narrowest supported scope.
- [ ] Functions host storage and trigger/binding permissions are validated separately.
- [ ] Required operations succeed and unnecessary operations fail in authorized tests.
- [ ] Old credentials are removed after cutover; confirmed exposed credentials are revoked or rotated.

## The Leak

- [ ] Storage account and container settings enforce intended data access.
- [ ] No anonymous access exists without an approved public-data requirement.
- [ ] A clean request to the unsigned test URL cannot retrieve private data.
- [ ] Legitimate application access still works after the change.
- [ ] Available logs and potentially exposed content are reviewed when exposure was demonstrated.

## The Door

- [ ] Every required inbound path has a documented caller and purpose.
- [ ] Unnecessary public access is denied using the approved network configuration.
- [ ] The unmatched-traffic action is verified explicitly.
- [ ] Main-site and SCM/Kudu access are reviewed separately.
- [ ] Authorized tests confirm blocked public access and successful required operations.

## The Call

- [ ] Findings distinguish observations, inference, and demonstrated behavior.
- [ ] Priorities account for active exposure, exploitability, impact, and workload purpose.
- [ ] Each correction has an owner and measurable closure evidence.
- [ ] Promotion waits for closure or a documented, approved exception.
- [ ] The release record retains the baseline version, reviewer, and validation evidence.
