# Fix LCM Provisioning workflows (approvalSummaries NPE)

## Problem
With `approvalScheme=none`, IdentityRequest often has `approvalSummaries=null`.  
Perform Identity Request Maintenance then fails:

```text
LCM request scan failed ... approvalSummaries is null
```

## Workflow to update
Primary: **LCM Provisioning - Without Approvals**  
Also apply to **LCM Provisioning** if it also uses `approvalScheme=none`.

---

## Change 1 — Ensure empty approvalSummaries after Initialize

### 1a) Add a new Step (insert after Initialize, before Create Ticket)

Add this step XML:

```xml
  <Step icon="Task" name="Ensure Approval Summaries" posX="260" posY="10">
    <Script>
      <Source>
        import sailpoint.object.IdentityRequest;
        import java.util.ArrayList;

        // identityRequestId is the request name (e.g. 0000001919)
        if (identityRequestId == null) {
          return "skipped: no identityRequestId";
        }

        IdentityRequest ir = context.getObjectByName(IdentityRequest.class, identityRequestId);
        if (ir == null) {
          return "skipped: IdentityRequest not found: " + identityRequestId;
        }

        if (ir.getApprovalSummaries() == null) {
          ir.setApprovalSummaries(new ArrayList());
          context.saveObject(ir);
          context.commitTransaction();
          return "fixed approvalSummaries for " + identityRequestId;
        }

        return "ok approvalSummaries already set for " + identityRequestId;
      </Source>
    </Script>
    <Transition to="Create Ticket"/>
  </Step>
```

### 1b) Rewire Initialize happy-path transition

Find in **Initialize** step the transition that currently goes to `Create Ticket` and change it to `Ensure Approval Summaries`.

**Before:**
```xml
    <Transition to="Create Ticket"/>
```

**After:**
```xml
    <Transition to="Ensure Approval Summaries"/>
```

Keep the Exit On Manual Work Items / Provisioning Form / Policy Violation transitions unchanged.

Flow becomes:

```text
Initialize → Ensure Approval Summaries → Create Ticket → Approve and Provision → ...
```

---

## Change 2 — Enable auto-verify on Finalize (recommended for STG)

In **Finalize** step, replace empty:

```xml
    <Arg name="autoVerifyIdentityRequest"/>
```

with:

```xml
    <Arg name="autoVerifyIdentityRequest" value="true"/>
```

This helps requests leave **Verifying** sooner when Get Object can confirm the change.

---

## Change 3 — Keep these settings (Without Approvals)

Already good in **LCM Provisioning - Without Approvals**:

| Variable | Value | Why |
|----------|-------|-----|
| `approvalScheme` | `none` | No approvals (your intent) |
| `optimisticProvisioning` | `true` | Apply entitlement changes to cube sooner |
| `foregroundProvisioning` | `true` | Provision in request thread for testing |

For Hoop Link `groups` refresh, still keep **AfterProvisioningRule** on the Hoop application. Optimistic provisioning alone is often not enough for Web Services.

---

## After updating workflow

1. Save/import the updated workflow XML.  
2. Patch **old** broken requests (`0000001690` …) with the fix rule (empty `approvalSummaries`).  
3. Re-run **Perform Identity Request Maintenance**.  
4. Submit a new Hoop Add/Remove request and confirm:
   - no `approvalSummaries is null` errors
   - request leaves Verifying
   - AfterProvisioning log refreshes groups

---

## Optional Debug check on one request

Open IdentityRequest `0000001690` in Debug. After fix you should see something like:

```xml
<entry key="approvalSummaries">
  <value>
    <List/>
  </value>
</entry>
```

Not missing / null.
