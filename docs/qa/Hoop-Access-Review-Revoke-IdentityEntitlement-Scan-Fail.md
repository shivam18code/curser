# Access Review revoke → LCM scan fails on `spt_identity_entitlement` delete

## Symptom

After **Access Review** (certification) **Revoke** of a Hoop entitlement on one identity, you run a **performance / Identity Request Maintenance** task and see:

```text
LCM request scan failed on request 0000001889 with exception
Batch update returned unexpected row count from update [0];
actual row count: 0; expected: 1;
statement executed: delete from spt_identity_entitlement where id=?
```

Provisioning / revoke may already have succeeded on Hoop. The failure is in **Perform Identity Request Maintenance** (PIRM / LCM request scan), not necessarily in the Hoop connector call.

## What the SQL means

Hibernate expected to **DELETE one row** from `spt_identity_entitlement` by primary key. The database returned **0 rows deleted** — the row was already gone (or never present under that id).

So the scan is trying to remove an `IdentityEntitlement` that another transaction already removed.

## Typical race (your sequence)

```text
1. Access Review decision: Revoke entitlement
2. Remediation creates IdentityRequest (e.g. 0000001889) → Remove Entitlement on Hoop
3. Optimistic provisioning / AfterProvisioning / Link update
   → IdentityEntitlement row removed (or rebuild drops it)
4. You run Performance / Identity Refresh / PIRM soon after
   → Refresh rebuilds IdentityEntitlements from Link / assignments
   → OR another path already deleted the same IE row
5. LCM request scan for 0000001889 tries: DELETE spt_identity_entitlement WHERE id=?
   → row count 0 → StaleStateException → "LCM request scan failed"
```

| Actor | What it does to IdentityEntitlement |
|-------|-------------------------------------|
| Access Review remediation + optimistic provisioning | Often removes IE from the cube when revoke is applied |
| AfterProvisioningRule (`getObject` → Link `groups`) | Updates Link; later refresh syncs IE from Link |
| Identity Refresh / aggregation | Rebuilds IE set from Link / assigned roles |
| PIRM LCM request scan | Tries to finalize request and delete the IE it still has in memory |

Two writers on the same IE id → second delete fails with **expected: 1, actual: 0**.

This is **not** the same as the earlier `approvalSummaries is null` NPE, but it shows up in the **same task** (LCM request scan / PIRM).

## Is the revoke “broken”?

Usually **no** for the target system:

1. Open IdentityRequest **0000001889** in Debug / UI.
2. Check provisioning status / completion (Success vs Verifying vs Failed).
3. On the identity cube → Accounts → **Hoop** → `groups`: is the revoked group gone?
4. Optional: `GET /api/users/{email}` — does Hoop match the Link?

| Cube / Hoop | Request scan | Meaning |
|-------------|--------------|---------|
| Group gone | Scan fails | Revoke worked; finalize/delete raced — **stale request cleanup** |
| Group still present | Scan fails | Fix Hoop Remove + AfterProv / re-run revoke; then clean request |
| Group gone | Scan OK later | Transient race; avoid overlapping refresh next time |

## Immediate fix for stuck request `0000001889`

If revoke already succeeded on Hoop/Link, mark the request **verified + completed** so PIRM **skips** it (same pattern as old `approvalSummaries` stuck requests).

### Debug rule (Beanshell) — one request

```beanshell
import sailpoint.object.IdentityRequest;
import sailpoint.object.IdentityRequest.ExecutionStatus;
import sailpoint.object.IdentityRequest.CompletionStatus;
import java.util.Date;

String n = "0000001889";
IdentityRequest ir = context.getObjectByName(IdentityRequest.class, n);
if (ir == null) {
  return n + " NOT_FOUND";
}

Date now = new Date();
ir.setVerified(now);
if (ir.getEndDate() == null) {
  ir.setEndDate(now);
}
ir.setExecutionStatus(ExecutionStatus.Completed);
ir.setCompletionStatus(CompletionStatus.Success);

context.saveObject(ir);
context.commitTransaction();
context.decache(ir);
return n + " COMPLETED verified=" + ir.getVerified();
```

Then re-run **Perform Identity Request Maintenance**. That request should no longer throw the delete error.

**Only do this after you confirm** the entitlement is actually gone on Hoop / Link. If it is still present, finish the revoke first — do not paper over a failed remove.

## Operational workaround (avoid the race)

1. Complete Access Review revoke remediation and wait until the IdentityRequest is **Finished / Success** (or at least provisioning committed).
2. **Then** run Identity Refresh / Performance / aggregation for that population.
3. Do **not** run PIRM + Identity Refresh on the same identity in parallel while remediation is still **Verifying**.
4. For STG testing: prefer one identity, one revoke, wait for LCM to settle, then refresh.

Optional PIRM task attribute (SailPoint community workaround for noisy pre-refresh failures):

```xml
<entry key="disablePreRefresh" value="true"/>
```

Use carefully; it skips PIRM’s pre-scan refresh, it does not fix a permanently stuck request. Prefer marking `verified` on the bad request.

## Relation to Hoop config

| Setting / component | Role in this error |
|---------------------|--------------------|
| `optimisticProvisioning=true` | Applies revoke to cube early → IE may already be deleted before scan |
| AfterProvisioningRule Link refresh | Good for `groups` on Link; then Refresh rebuilds IE — can race with scan delete |
| Get Object / aggregation | Rebuilds Link → IE sync; can delete IE the scan still expects |
| BeforeOperation Remove | Unrelated to this SQL; only affects Hoop PUT |

No Hoop connector change is required **just** for this Hibernate message. Fix timing + clear the stuck IdentityRequest.

## Checklist

1. Confirm request **0000001889** identity + which Hoop group was revoked.
2. Confirm Hoop API + Link `groups` already match the revoke.
3. Mark request verified/Completed if revoke succeeded; re-run PIRM.
4. Next Access Review revoke: wait for LCM finish before Performance / Refresh.
5. If scan fails again on a **new** request while group is still on Link → debug Remove Entitlement / AfterProv, not only PIRM.

## Related docs

- `docs/qa/Fix-IdentityRequest-ApprovalSummaries-NPE.md` — skip stuck requests via `verified`
- `docs/qa/LCM-Workflow-ApprovalSummaries-Fix.md` — PIRM / LCM scan context
- `docs/qa/Hoop-Groups-Refresh-After-Provisioning.md` — Link `groups` after Add/Remove
