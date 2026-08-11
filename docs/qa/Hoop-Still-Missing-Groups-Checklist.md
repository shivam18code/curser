# Hoop — Why groups still missing (check your live Application)

Your latest exported Application still has these problems. **AfterProvisioning cannot refresh groups until #1 is fixed.**

## Must fix in IIQ UI now

### 1) Get Object URL — still wrong in your export

You still have:
```text
/api/users/$plan.nativeIdentity$
```

Change **Single Account Aggregation (Get Object)** to:
```text
/api/users/$getObject.nativeIdentity$
```

Then **Save** the application.

Without this, `connector.getObject()` in AfterProvisioning fails/returns empty, so the cube stays stale.

### 2) Add / Remove still have empty response mapping

Your export shows Add Entitlement and Remove Entitlement with **no** `resMappingObj`.

On each of: **Add Entitilement**, **Remove Entitlement**, **Enable Account**, **Disable Account**, **Create Account**:

| Field | Value |
|-------|--------|
| Response Root Path | `$` |
| Map schema attrs | `id←id`, `name←name`, `email←email`, `status←status`, `role←role`, `groups←groups` |

### 3) Re-import AfterProvisioningRule

Import updated `Amlak-Rule-Hoop-AfterProvisioningRule.xml` (now has **plan-based groups fallback** if getObject fails).

Confirm Application → **After Provisioning Rule** = `Amlak-Rule-Hoop-AfterProvisioningRule`.

## Verify after save

1. Add a group to a test user.
2. Do **not** run manual aggregation.
3. Check logs `com.amlak.application.hoop` for one of:
   - `Refreshed Link from getObject ... groups=...`
   - `Fallback refreshed Link groups from plan ... groups=...`
4. Open Identity Cube → Hoop account → groups should match.

If you still see:
`getObject refresh failed ...` or `getObject returned null`
→ Get Object URL is still `$plan...` — fix step 1 and save again.

## Quick checklist vs your paste

| Item | Your paste | Required |
|------|------------|----------|
| Get Object URL | `$plan.nativeIdentity$` ❌ | `$getObject.nativeIdentity$` |
| Add Entitlement response mapping | empty ❌ | map groups/status/… |
| Remove Entitlement response mapping | empty ❌ | map groups/status/… |
| After Provisioning Rule | set ✅ | re-import updated rule |
| Add/Remove body `groups` only | ✅ | keep |
