# Hoop LCM log analysis — only `ba` shows after refresh

## Symptom

From LCM / workflow traces (IdentityRequests `0000001921`–`0000001926` and related):

- Several Hoop operations report **`ProvisioningResult status="committed"`** and IdentityRequests complete as **Success**.
- After **Identity Refresh** / performance tasks, the Identity Cube often shows **only the `ba` group** (or other access disappears).
- User report: *“from this only add ba group showing after refresh performance tasks all other operations worked but not showing”*.

This document explains what the pasted logs prove, what they do **not** prove, and how the repo fixes address Create / implicitCreate email failures and Link refresh.

---

## What the logs show (by request)

| IdentityRequest | Identity | Operation | Result in log |
|-----------------|----------|-----------|---------------|
| `0000001921` | `hmarko` | Add `groups=ba` (Modify) | **Success** — `provisioningState="Finished"`, verified |
| `0000001922` | `rtestpatil` | Add `groups=admin` → **Create** (`implicitCreate`) | **Failure** — Create POST without email |
| `0000001923` | `vkamble.v` | Create Account | Committed, then **Verifying** (`autoVerifyIdentityRequest` null on plain LCM Provisioning) |
| `0000001924` | `vkamble.v` | Add `groups=ba` | **Success** |
| `0000001925` | `vkamble.v` | Remove `groups=ba` | **Success** (committed) |
| `0000001926` | `vkamble.v` | Add `groups=db` | **Success** (committed) |

Grafana CustomizationRule lines (`krao@amlakint.com`, `IIQDisabled`) are **unrelated** noise from another app on Quartz workers.

Long **Wait for Entra Aggregation** / **Waiting For Manual Action On Work Item** loops are separate workflow steps; they do not explain Hoop group display by themselves.

---

## Failure that is clear in the log: Create / implicitCreate without email

### `0000001922` (`rtestpatil`)

Plan started as Modify Add `admin` with:

```xml
<AccountSelection ... implicitCreate="true"/>
```

Compiled to **Create** with only:

- `groups=admin`
- `displayName` / `name` from ProvisioningPolicy

**No `email` AttributeRequest.** Native identity stayed null for the connector call.

Hoop API response:

```text
POST https://hoopui-stg.pfa.amlakint.com/api/users
400 Bad Request
{"message":"Key: 'User.Email' Error:Field validation for 'Email' failed on the 'required' tag"}
```

Workflow message:

```text
Exception occurred while performing 'Create' operation on identity 'null':
Url: .../api/users, Message: 400 : Bad Request : ... 'Email' failed on the 'required' tag
```

### Root cause (IIQ + BeforeRule)

1. Create Account body must include **email**. Relying only on `$plan.email$` fails when Create Form / policy does not put email on the plan (common on **implicitCreate** from Access Request).
2. BeforeOperation rule was attached to Create Account. For `POST /api/users` (collection URL, **no** email path segment), an older rule treated the path segment `users` as “email”, GETs the wrong URL, then **rewrote** `jsonBody` and wiped a correct `$plan.email$` body.

### Repo fixes

1. **`Hoop.xml` Create Form** — email Field Script always resolves identity email / mail / UPN / name-with-`@`.
2. **Create Account `jsonBody`** — `{"email":"$plan.email$","name":"$plan.displayName$","status":"active"}`.
3. **`Amlak-Rule-BeforeRule-Hoop`** — on Create (or endpoint name containing `create`):
   - Resolve email from AttributeRequest / Identity attributes / `identity.getEmail()` / name-with-`@`
   - Set `AccountRequest.nativeIdentity` to that email
   - Ensure body has email / name / status
   - **Do not** run Get-Object merge / path-as-email logic on Create

Re-import **Application** + **BeforeRule**, then retry Create and Access Request → implicitCreate.

---

## Why “committed” does not mean the cube will show groups after refresh

### Optimistic provisioning

Several runs use:

```text
optimisticProvisioning = true
doRefresh = false
```

and skip **Refresh Identity**.

With optimistic provisioning, IIQ can mark IdentityRequest items **Finished** and show temporary entitlement state **without** confirming the Link attribute from the target. After Identity Refresh / aggregation, the cube is rebuilt from what aggregation / Get Object returns.

So:

- Workflow success ≠ durable Link.groups
- Refresh can remove optimistic rows that were never written to the Link (or never present in Hoop)

### AfterProvisioningRule must refresh Link.groups

For Web Services Modify, if Add/Remove Entitlement response mapping is empty, **Get Object does not automatically update the Link**. The app must run **`Amlak-Rule-Hoop-AfterProvisioningRule`** after committed Create/Modify/Enable/Disable to:

1. Prefer `connector.getObject` → set Link `groups` / `status` / etc.
2. Else merge plan AttributeRequests for `groups` onto the Link

**Important:** The pasted LCM / Quartz excerpt has **no** lines from logger `com.amlak.application.hoop` (AfterProv / BeforeRule). That strongly suggests the AfterProvisioning rule was **not imported / not attached / not executing** on that IIQ instance when these requests ran.

Without AfterProv:

- Cube can look “updated” briefly under optimistic provisioning
- After Refresh / aggregation, only attributes returned by Hoop aggregation remain
- If Hoop still only has `ba` (or aggregation only maps `ba`), the cube shows only `ba`

### What to check on the environment

1. Application **Hoop** → After Provisioning Rule = `Amlak-Rule-Hoop-AfterProvisioningRule` (imported).
2. During Add/Remove Entitlement, logs must show AfterProv lines (`AfterProv`, `getObject`, `setAttribute groups`, etc.) under `com.amlak.application.hoop`.
3. Manual **GET** `https://hoopui-stg.pfa.amlakint.com/api/users/{email}` — does `groups` contain `ba`, `db`, `admin`, etc.?
4. **Get Object** operation URL uses `$getobject.nativeIdentity$` and maps `groups`.
5. Single Account Aggregation for that user — Link.groups after aggregation should match the API.

If the API never received Add `db` / Remove `ba`, fix BeforeRule merge + Create email first. If the API has the groups but the cube does not after refresh, fix AfterProv + aggregation.

---

## `vkamble.v` sequence vs “only ba after refresh”

From the logs:

1. Create Account committed (`0000001923`) — may still be Verifying under plain LCM Provisioning.
2. Add `ba` committed + Success (`0000001924`).
3. Remove `ba` committed + Success (`0000001925`).
4. Add `db` committed + Success (`0000001926`).

If after performance refresh the cube only shows `ba`:

| Possibility | Meaning |
|-------------|---------|
| AfterProv never ran | Optimistic UI lied; Link never updated; aggregation shows target truth |
| Target still has only `ba` | BeforeRule / API merge failed for later ops; refresh correctly shows only `ba` |
| Aggregation maps incompletely | Get Object / group schema mapping wrong |
| Looking at wrong UI surface | IdentityEntitlement vs Link attribute `groups` (Accounts → Hoop → attributes) |

**Do not** treat IdentityRequest Success alone as proof the cube Link was updated.

---

## Create Account verification / Entra wait (secondary)

- Plain **LCM Provisioning** left `autoVerifyIdentityRequest` null → request stayed **Verifying** even after committed Create.
- Prefer **LCM Provisioning - Without Approvals** (or set auto-verify) for lab Create tests.
- Entra aggregation wait retries returning false are a **different** step; they do not replace Hoop Single Account Aggregation for Link.groups.

---

## Recommended verification checklist (after importing fixes)

1. Import `Hoop.xml`, `Amlak-Rule-BeforeRule-Hoop`, `Amlak-Rule-Hoop-AfterProvisioningRule`.
2. Confirm AfterProv is attached on the Application.
3. Create Account for an identity with email → expect 201 and nativeIdentity = email (not null / not `users`).
4. Access Request Add group on identity **without** Hoop account → implicitCreate must send email (BeforeRule Create path).
5. Add group on existing account → AfterProv log + Link.groups includes new value **before** refresh.
6. Remove group → AfterProv log + Link.groups updated **before** refresh.
7. Run Identity Refresh / aggregation → Link.groups still matches Hoop GET (not “only ba” unless API only has ba).

---

## Related docs

- [Hoop-Groups-Refresh-After-AddRemove.md](./Hoop-Groups-Refresh-After-AddRemove.md)
- [Hoop-WebServices-GetObject-vs-AfterProvisioning.md](./Hoop-WebServices-GetObject-vs-AfterProvisioning.md)
- [Hoop-Create-Account-Email-Required-Fix.md](./Hoop-Create-Account-Email-Required-Fix.md)
- [Hoop-QA-Verification-Checklist.md](./Hoop-QA-Verification-Checklist.md)
