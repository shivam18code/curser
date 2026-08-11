# Hoop — Groups not visible after Add/Remove (fix guide)

## Symptom

After **Add Entitlement** or **Remove Entitlement** succeeds in IIQ, the Identity Cube / account **groups** field still shows the old value. Groups only update after you manually run **Single Account Aggregation** (Get Object) for that user.

## Why it happens

1. Hoop PUT succeeds on the target.
2. IIQ **Link** attributes (`groups`) are **not automatically refreshed** after Modify for Web Services unless:
   - the operation response is mapped back into schema attributes, **or**
   - an AfterProvisioning / AfterOperation rule refreshes the Link.
3. Your previous Add/Remove endpoints had **empty `resMappingObj`**, so even if Hoop returned the user JSON, IIQ ignored it.
4. Your AfterProvisioningRule only patched **`status`** on Enable/Disable — it did **not** refresh **`groups`** after Modify.

Manual aggregation works because Get Object reads Hoop and updates the Link — that is exactly what was missing after provisioning.

## Fix (do both)

### 1) Application — response mapping on Modify endpoints

On **Add Entitlement**, **Remove Entitlement**, **Enable Account**, **Disable Account** (and ideally **Create Account**):

| Setting | Value |
|---------|--------|
| Response Root Path | `$` |
| Schema attribute mapping | `id`, `name`, `email`, `status`, `role`, `groups` → same JSON keys |

Only helps if Hoop’s PUT/POST **returns** the full user object in the response body. If Hoop returns empty/`204`, mapping alone is not enough — step 2 covers that.

Repo file: `iiq/applications/Hoop.xml` (already includes these mappings).

### 2) AfterProvisioningRule — refresh Link via getObject

Import/update `Amlak-Rule-Hoop-AfterProvisioningRule` and keep it set as the application **After Provisioning Rule**.

After successful **Create / Modify / Enable / Disable** it:

1. Calls `connector.getObject("account", nativeIdentity)` (same as Single Account Aggregation).
2. Copies `groups`, `status`, `name`, `email`, `role`, `id` onto the Link.
3. Saves/commits the Link.
4. Enable/Disable: if getObject fails, still falls back to patching `status`.

Repo files:

- `iiq/rules/Amlak-Rule-Hoop-AfterProvisioningRule.xml` ← import this
- `iiq/rules/Amlak-Rule-Hoop-AfterProvisioningRule.bsh` ← review mirror only

## Deploy steps

1. Import `Amlak-Rule-Hoop-AfterProvisioningRule.xml` (Debug → Import or iiq console).
2. Application **Hoop** → Configuration:
   - After Provisioning Rule = `Amlak-Rule-Hoop-AfterProvisioningRule`
   - On Add/Remove/Enable/Disable endpoints: set response mapping as above (or re-apply from `Hoop.xml`).
3. Save application.
4. Confirm logger `com.amlak.application.hoop` is INFO.
5. Test Add then Remove for a known user **without** manual aggregation.

## Verify

| Check | Expected |
|-------|----------|
| Provisioning transaction | Committed / Success |
| Cube → Accounts → Hoop → groups | Updated immediately after request completes |
| Logs | `Refreshed Link from getObject for <email> op=Modify groups=...` |
| Hoop API `GET /api/users/{email}` | Matches IIQ Link groups |

If logs show getObject failure, fix Get Object endpoint / OAuth first (manual Single Account Aggregation would also fail).

## Get Object URL token (important)

| Operation | Context URL identity token |
|-----------|----------------------------|
| Get Object / Single Account Aggregation | **`$getObject.nativeIdentity$`** |
| Create / Add / Remove / Enable / Disable | **`$plan.nativeIdentity$`** |

Do **not** put `$plan.nativeIdentity$` on Get Object. Manual aggregation and AfterProvisioning `connector.getObject()` use the **`$getObject`** map. With `$plan` there, Get Object can fail or hit the wrong user, so Link groups stay stale.

Example:
```text
Get Object → GET /api/users/$getObject.nativeIdentity$
Add Entitlement → PUT /api/users/$plan.nativeIdentity$
```

## What not to rely on

| Setting | Why it does not fix this |
|---------|---------------------------|
| `isGetObjectRequiredForPTA=true` | Used for plan evaluation **before** provisioning, not Link refresh after |
| Manual aggregation forever | Workaround only |
| BeforeOperation rule | Rewrites outbound PUT body; does not update IIQ Link |
| `$plan.nativeIdentity$` on Get Object | Wrong placeholder for getObject / Single Account Aggregation |

## Quick test cases

1. User has `G-A` only → Add `G-B` → open cube **without** aggregating → both groups visible.
2. User has `G-A`, `G-B` → Remove `G-B` → cube shows only `G-A` without aggregating.
3. Disable → status `inactive` on Link without aggregating.
4. Enable → status `active` on Link without aggregating.
