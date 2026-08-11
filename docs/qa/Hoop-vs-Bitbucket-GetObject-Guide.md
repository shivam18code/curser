# Hoop vs Bitbucket — why Bitbucket groups update, Hoop does not

## Short answer

Bitbucket **also** has empty Add/Remove response mapping. That is **not** why it works.

Bitbucket works because:

1. **Get Object is correct** — uses `$getobject.nativeIdentity$` (not `$plan...`)
2. **Get Object returns `groups`** — second Get Object endpoint maps `groups`
3. It also has **AfterProvisioningRule** (same idea as Hoop) — so this is **not** proof that Web Services auto-calls Get Object after Modify with no AfterProvisioning

Web Services does **not** auto-refresh the Link from Add/Remove when those ops have no response body/mapping. Bitbucket matches that pattern.

---

## Side-by-side

| Item | PROD Bitbucket (works) | Hoop (stale groups) |
|------|------------------------|---------------------|
| Add Entitlement response map | **Empty** | **Empty** |
| Remove Entitlement response map | **Empty** | **Empty** |
| Get Object identity token | **`$getobject.nativeIdentity$`** | Your export still had **`$plan.nativeIdentity$`** |
| Get Object returns groups? | Yes (child endpoint maps `groups`) | Yes **if** Get Object hits user API and maps `groups` |
| AfterProvisioningRule | `Amlak-Rule-Bitbucket-AfterProvisioningRule` | `Amlak-Rule-Hoop-AfterProvisioningRule` |
| `isGetObjectRequiredForPTA` | true | true |

So copying Bitbucket means: **fix Get Object**, not “add PUT response mapping.”

---

## What Bitbucket Get Object looks like (the important part)

**Endpoint 1 — account**
```text
GET /2.0/workspaces/.../members/$getobject.nativeIdentity$
```
Maps account fields (`account_id`, `displayName`, …).

**Endpoint 2 — groups (parent = endpoint 1)**
```text
GET .../users?expand=groups&accountIds=$response.account_id$...
```
Maps:
- `groups` ← `$.groups[*].id`
- `groupName`, `status`

When Single Account Aggregation / `getObject` runs, **groups are loaded**.

---

## What Hoop Application XML should look like (Bitbucket-style)

Hoop API already returns groups on the user object, so you only need **one** Get Object endpoint (simpler than Bitbucket).

### Change this only (main Application XML fix)

**Get Object / Single Account Aggregation**

| Field | Wrong (your Hoop export) | Correct (like Bitbucket) |
|-------|--------------------------|---------------------------|
| Context URL | `/api/users/$plan.nativeIdentity$` | `/api/users/$getobject.nativeIdentity$` |
| Root path | `$` | `$` |
| Response map | `id,name,email,status,role,groups` | keep as-is (must include **`groups`**) |

Use the same token style as Bitbucket: `$getobject.nativeIdentity$`.

### Leave like Bitbucket (OK)

- Add Entitlement: **no** response mapping  
- Remove Entitlement: **no** response mapping  
- Bodies that only send what the API needs  

Do **not** expect Add/Remove alone to refresh the cube.

---

## About “without AfterProvisioning it should call Get Object”

That assumption is **not** how Bitbucket is set up either:

- Bitbucket Add/Remove do not return mapped account+groups
- Bitbucket **has** AfterProvisioningRule
- Bitbucket **does** have a working Get Object that includes groups

So:

| Mechanism | Bitbucket | Hoop should do |
|-----------|-----------|----------------|
| Modify response mapping | No | No (optional only if Hoop PUT returns full user) |
| Correct Get Object | Yes | **Yes — fix URL token** |
| AfterProvisioning refresh | Yes | Yes (same pattern), once Get Object works |

If you temporarily remove AfterProvisioning on Bitbucket and Add/Remove still update the cube, that would be IIQ updating entitlements from the **plan** on commit — then Hoop would need the same successful commit path. In practice your Bitbucket app is built around **Get Object + AfterProvisioning**, not Modify response mapping.

---

## Exact Hoop checklist (Application XML only)

1. Open Hoop → HTTP Operations → **Single Account Aggregation** / Get Object  
2. Set URL to:
   ```text
   /api/users/$getobject.nativeIdentity$
   ```
3. Confirm response map includes **`groups` ← `groups`** (and status/name/email/id/role)  
4. Root path `$`  
5. **Save** application  
6. Test **manual** Single Account Aggregation on one user — groups must appear.  
   - If manual Get Object fails/wrong user → AfterProvisioning refresh will also fail  
   - If manual Get Object works → Application Get Object is fixed; then AfterProvisioning can refresh after Add/Remove like Bitbucket  

---

## Do not copy from Bitbucket into Hoop

- Bitbucket 2-step Get Object (Hoop is 1-step)  
- Bitbucket membership POST/DELETE URLs  
- Bitbucket AfterOperation rules for aggregation  
- Any passwords/tokens from the Bitbucket export
