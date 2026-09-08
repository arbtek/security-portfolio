# The Stolen Identity

## How one phished employee became an application problem

> **Scenario:** Reconstructed a five-stage OAuth consent-phishing kill chain in a live Azure tenant through forensic analysis of two linked app registrations.  
> **Access:** Reader on directory applications  
> **Status:** Investigation complete  
> **Publication boundary:** No flags, secrets, tenant IDs, application IDs, exact scope values, or attacker redirect URLs are included.

## The short version

The attacker did not break through Azure with some exotic zero-day. They phished an employee from accounting.

The employee entered a password into a convincing fake login page and completed MFA. The attacker stole the resulting session token. MFA worked; the attacker stole the result (This bypasses Conditional Access Policies). This lets the attacker sign in normally without triggering alarms that alert the security team. 

That would already have been bad enough, but the employee was still an owner of a legacy enterprise application. The assignment had survived long after the business reason for it had apparently expired. The attacker used those owner rights to create a new client secret, authenticate as the application's service principal, and inherit its live Microsoft Graph application permissions.

Then they made the access harder to remove. They created a second app registration, added its service principal as an owner of the legacy application, exposed a custom API scope, and configured a redirect URI pointing to attacker-controlled infrastructure.

The attack moved through five stages:

**Entry → Escalate → Pivot → Persist → Loot**

Or, less politely:

**Steal the session → become the app → own the app → make the app callable → collect the token**

## Objective 1: Entry

### Where I looked

I opened the legacy application's **Branding & properties** blade and reviewed the internal notes.

### What I found

The first foothold was a real employee from accounting. The user was sent to a convincing fake login page, entered a password, and completed MFA. The attacker captured the authenticated session token and reused the MFA-satisfied session.

The account was not a Global Administrator. It did not need to be. It was still an owner of the legacy application.

That is the first important distinction in this investigation: directory role and application ownership are not the same thing. A review of Global Administrators could have come back perfectly clean while the useful privilege sat elsewhere, quietly aging in the Owners list.

The password got the attacker to the door. The stale owner assignment handed over the keys.

<img width="1310" height="896" alt="entry" src="https://github.com/user-attachments/assets/5ac5be02-a1a5-476e-8648-9211e0952c30" />

## Objective 2: Escalate

### Where I looked

I moved to **API permissions**, followed by **Certificates & secrets**.

### What I found

The legacy application had live Microsoft Graph application permissions with admin consent. The visible permissions included `Directory.Read.All` and `User.Read.All`.

Application permissions matter because the app acts on its own behalf. No human user is involved in the client-credentials flow. If the application can prove its identity with a valid credential, Entra issues an app-only token containing the permissions already granted to that service principal.

The attacker used the compromised employee's owner rights to add a client secret directly to the legacy app. Under **Certificates & secrets**, I found the new credential. Its expiration date was set in 2099.

A secret expiring in 2099 is not a serious expiration policy. It is optimism formatted as a date.

With that secret, the attacker no longer needed to keep signing in as the phished employee. They could authenticate programmatically as the service principal and request app-only tokens whenever needed.

This was the escalation: a compromised standard user gained control over a privileged workload identity.

<img width="1495" height="694" alt="api_permissions" src="https://github.com/user-attachments/assets/721212c0-2ec7-4510-bbac-df12fbc7f3a0" />

*The legacy application's Microsoft Graph permissions were application permissions and had admin consent. This made the permissions live and defined the blast radius of the stolen app credential.*

<img width="1308" height="716" alt="escalate" src="https://github.com/user-attachments/assets/ad4c6a54-418b-4a37-baec-d7f32a5e99f2" />

*The legacy application's client secret was set not to expire until the end of the century.*

## Objective 3: Pivot

### Where I looked

I reviewed the legacy application's **Owners** list, then opened the second app registration and checked its properties.

### What I found

The attacker had created a new app registration and added its service principal as an owner of the legacy application.

The first client secret gave the attacker access. The ownership pivot lets them recover that access after partial cleanup.

If a defender found the original secret and removed it, the rogue owner could create another credential. Rotating one secret would solve the visible problem while leaving the authority that created it untouched. That is not remediation. It is mowing the weed and leaving the root.

This also exposed another dangerous default: standard users can register applications in Entra unless the tenant changes that setting. The feature exists for legitimate development. The attacker was not offended by its intended purpose.

The rogue app functioned as a shadow administrator over the legacy application. It would not appear in a directory-role review, because it did not require a privileged directory role. The privilege lived in the relationship between the two application objects.

<img width="1337" height="691" alt="pivot" src="https://github.com/user-attachments/assets/a1241329-ea68-4286-ae35-35d1bb67f53c" />

## Objective 4: Persist

### Where I looked

I opened **Expose an API** on the legacy application.

### What I found

The attacker had created a custom delegated scope. This is because the attacker understands that it is standard practice to rotate all secrets and passwords for all accounts/services implicated in a breach.

The **Expose an API** configuration tells Entra that an application can operate as a secured backend resource that other applications may request permission to call. In this case, the attacker had already created the other application.

The scope wasn't a magic token, and it didn't automatically inherit the legacy application's Graph permissions. It created a second authorization path. The rogue client could now request delegated access to the legacy application's backend through an OAuth consent flow.

That distinction matters. An exposed scope is a doorway, not proof that someone walked through it. The risk becomes severe if the trusted backend accepts the delegated request and then uses its own higher privilege without properly checking the caller, user, tenant, scope, and requested action.

The attacker was building a backup plan: if the app-only credential was discovered, the rogue application could launch a consent-phishing campaign against another signed-in employee.

<img width="1356" height="805" alt="expose_an_api" src="https://github.com/user-attachments/assets/1fd665a2-5e54-4a33-bac7-ddf43b3e5127" />

## Objective 5: Loot

### Where I looked

I opened the rogue application's **Authentication** blade and reviewed its redirect URIs.

### What I found

One redirect URI pointed to attacker-controlled infrastructure.

In the OAuth authorization-code flow, the redirect URI is where Microsoft sends the browser after authorization, with an authorization code attached. The attacker combined three things:

1. The rogue application's client ID
2. The attacker-controlled redirect URI
3. The custom scope exposed by the legacy application

That produced the consent-phishing URL.

The victim did not have to type a password into another obviously suspicious page. The victim could already be signed in on a managed corporate device and have satisfied MFA and device requirements. They would see a real Microsoft consent prompt. If they clicked **Accept**, Entra would send the authorization code to the attacker's registered redirect URI, where the rogue application could exchange it for a delegated access token.

The counterfeit part was the request. The login and consent machinery were genuine. This is why the technique is so effective: the attacker borrows the identity provider's credibility and invites the victim to authorize the theft personally.

## Why not just phish the password again?

Because a new login creates another opportunity for security controls to interfere.

A fresh attacker sign-in may fail because the device is unmanaged, the IP address is suspicious, the location is wrong, MFA is required again, or token protection prevents replay. Consent phishing asks a user who is already authenticated on a trusted device to approve an application instead.

The comforting fiction is that the password is the identity. It is not. Modern identity is a collection of sessions, applications, service principals, credentials, ownership relationships, permissions, scopes, redirect URIs, and grants. Resetting one password addresses exactly one part of that system.

When the victim approves delegated access, Entra creates an `OAuth2PermissionGrant`.

- Changing the password does not delete the grant.
- Revoking sign-in sessions does not delete the grant.
- Enforcing MFA does not delete the grant.

Those containment actions are still necessary for the compromised user. They simply do not remove the application's authorization. You must find and revoke the malicious grant separately.

Deleting the grant prevents new tokens from being issued under that consent. Access tokens already issued can remain valid until they expire. The exact response, therefore, is not “reset the password and declare victory.” It is:

> Reset the user. Revoke the sessions. Remove the app's access. Then verify that all three actions actually worked.

## The confused deputy

This attack follows the confused-deputy pattern.

The legacy connector is the deputy: a trusted application with legitimate authority. The rogue app is the caller. If the legacy backend accepts the delegated request and then performs a higher-privilege operation using its own application permissions without properly enforcing the authorization boundary, the attacker has persuaded a trusted service to exercise power on behalf of someone who should not possess it.

The deputy is not malicious. It is confused. Unfortunately, the directory does not award points for good intentions.

## Findings

### Critical: Unauthorized credential on a privileged application

The attacker added a client secret to a legacy application with admin-consented Microsoft Graph application permissions. The secret remained valid into 2099 and enabled non-interactive app-only authentication.

### Critical: Rogue service principal added as an owner

The attacker-controlled service principal could create replacement credentials and change application configuration after a defender removed the original secret.

### High: Illicit delegated-consent path

The custom scope, rogue client application, and attacker-controlled redirect URI formed a working authorization-code collection path.

### High: Excessive application permissions

The legacy application's broad directory permissions increased the blast radius of any credential or ownership compromise.

### Medium: Default user app registration remained enabled

The compromised user could create the rogue registration without first obtaining a privileged directory role.

### Medium: High-signal application changes were not detected

The attack required a new secret, a new owner, a new app registration, a new scope, a new redirect URI, and a consent grant. Each event was an opportunity to detect the operation. None produced an effective response.

## Recommendations

### Immediate containment

1. Disable or restrict the affected service principals while preserving the evidence needed for the investigation.
2. Remove the unauthorized client secret and rotate any other application credentials that may have been exposed.
3. Remove the rogue service principal from the legacy application's Owners list.
4. Remove the attacker-controlled redirect URI.
5. Delete the malicious `OAuth2PermissionGrant` explicitly.
6. Remove the unauthorized custom scope after checking for legitimate dependencies.
7. Reset the compromised user's password, revoke sessions, and review authentication methods.

### Scope the incident

1. Review user and service-principal sign-ins for both applications and every affected identity.
2. Search directory audit logs for credential additions, owner changes, app registrations, permission changes, redirect URI changes, scope changes, and consent activity.
3. Determine what directory data or backend operations the attacker accessed.
4. Inventory every credential, owner, permission grant, and app-role assignment associated with both applications.
5. Review the legacy backend's authorization logic to determine whether delegated access could invoke higher-privilege operations.

### Prevent it from happening again

1. Reduce the legacy application's Graph permissions to the least privilege required for its real business purpose.
2. Disable default user app registration where practical and grant the Application Developer role through an approved process.
3. Restrict user consent and use an admin-consent workflow for applications that need broader access.
4. Audit application Owners lists the same way privileged directory roles are audited.
5. Establish reasonable credential lifetimes and prefer certificates or workload identity federation over long-lived shared secrets where supported.
6. Alert on new credentials, unusually long expiration dates, owner additions, new app registrations, redirect URI changes, exposed scopes, permission grants, and consent events.

## What surprised me

The long-lived secret was obviously bad. The more interesting problem was the ownership relationship.

A standard user's stale owner assignment became control over a privileged workload identity. The attacker then used a second service principal to preserve that control. Nothing about the path required the compromised user to appear in a privileged directory role.

That changed how I think about identity reviews. It is not enough to ask who the administrators are. You also have to ask what they own, what owns what, which credentials those objects trust, and which permissions have already been granted.

The other surprise was how incomplete the familiar containment checklist can be. Password reset, session revocation, and MFA enforcement sound decisive because they address a narrow class of problems. They do not erase application consent. The grant sits in the directory, perfectly valid and entirely indifferent to the confidence with which someone closes the incident ticket.

## References

- [Microsoft identity platform: OAuth 2.0 client credentials flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant-flow)
- [Microsoft identity platform: Permissions and consent overview](https://learn.microsoft.com/en-us/entra/identity-platform/permissions-consent-overview)
- [Microsoft Entra ID: Protect against consent phishing](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/protect-against-consent-phishing)
- [Microsoft Security: App consent grant investigation](https://learn.microsoft.com/en-us/security/operations/incident-response-playbook-app-consent)
- [Microsoft Defender for Office 365: Detect and remediate illicit consent grants](https://learn.microsoft.com/en-us/defender-office-365/detect-and-remediate-illicit-consent-grants)
- [Microsoft Graph PowerShell: Remove-MgOauth2PermissionGrant](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.signins/remove-mgoauth2permissiongrant)
