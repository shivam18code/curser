# Hoop — current status after your latest Application export

## Application XML: Get Object is fixed

Your latest export has:

```text
Get Object → /api/users/$getObject.nativeIdentity$
```

with `groups` mapped. That matches the needed Application pattern (Bitbucket uses `$getobject` lowercase — use that exact token to be safe).

| Item | Status |
|------|--------|
| Get Object uses getObject token (not plan) | OK (prefer `$getobject.nativeIdentity$`) |
| Get Object maps `groups` | OK |
| Add/Remove empty response map | OK (same as Bitbucket) |

**Important:** Fixing Get Object alone does **not** make Add/Remove auto-refresh the cube. Bitbucket also needs its **AfterProvisioningRule** for that. Web Services does not call Get Object after Modify by itself.

---

## Why you still cannot see groups

Remaining cause is almost certainly:

1. **AfterProvisioningRule in IIQ is still the old version** (only patched Enable/Disable `status`), **or**
2. AfterProvisioning `getObject` fails / plan fallback does not run, **or**
3. You are looking at a cached Identity Cube tab (close/reopen identity)

Application XML is no longer the main blocker.

---

## Do this now (in order)

### 1) Match Bitbucket token exactly
Change Get Object URL to lowercase (like Bitbucket):

```text
/api/users/$getobject.nativeIdentity$
```

Save application.

### 2) Prove Get Object works
On a test user: **Accounts → Hoop → Aggregate Account** (Single Account Aggregation).

- If groups appear → Get Object is good.
- If not → Get Object still broken (URL/auth/mapping); stop here.

### 3) Import updated AfterProvisioningRule
Import `iiq/rules/Amlak-Rule-Hoop-AfterProvisioningRule.xml` from this repo (overwrite existing).

Confirm Application → After Provisioning Rule = `Amlak-Rule-Hoop-AfterProvisioningRule`.

### 4) Retest Add or Remove (no manual agg)
Watch log `com.amlak.application.hoop` for one of:

```text
Refreshed Link from getObject ... groups=...
Fallback refreshed Link groups from plan ... groups=...
```

If you see:

```text
getObject failed ...
Link NOT refreshed ...
```

paste that log line — that is the next fix.

### 5) Reopen the Identity Cube
Close the identity and open again (or refresh entitlements view). Stale browser tab often shows old groups.

---

## What Application XML does vs does not do

| Action | Updates cube groups after Add/Remove? |
|--------|----------------------------------------|
| Correct Get Object URL/mapping | Only when Get Object / AfterProv actually runs |
| Empty Add/Remove response map | No (same as Bitbucket) |
| AfterProvisioning getObject/plan fallback | Yes — this is what Bitbucket-style setup relies on |
