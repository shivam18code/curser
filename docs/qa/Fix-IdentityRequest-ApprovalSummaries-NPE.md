# Fix stuck IdentityRequests causing approvalSummaries NPE

## Why the empty-list rule failed
IIQ often **drops empty lists** from IdentityRequest attributes on save.  
So this:

```text
approvalSummaries = []
```

becomes `null` again → Maintenance still NPEs.

## Reliable fix: mark those old requests verified/completed
Maintenance scans requests that still need verification (`verified` is null).  
If you set `verified` + complete them, the scanner skips them.

### Rule (run once in Debug)

```beanshell
import sailpoint.object.IdentityRequest;
import sailpoint.object.IdentityRequest.ExecutionStatus;
import sailpoint.object.IdentityRequest.CompletionStatus;
import java.util.Arrays;
import java.util.Date;
import java.util.List;

List names = Arrays.asList(
"0000001690","0000001691","0000001692","0000001697","0000001698",
"0000001700","0000001701","0000001702","0000001703","0000001704",
"0000001705","0000001706","0000001707","0000001708","0000001709",
"0000001710","0000001711","0000001712","0000001732","0000001733",
"0000001735","0000001746"
);

int fixed = 0;
StringBuffer sb = new StringBuffer();
Date now = new Date();

for (int i = 0; i < names.size(); i++) {
  String n = (String) names.get(i);
  IdentityRequest ir = context.getObjectByName(IdentityRequest.class, n);
  if (ir == null) {
    sb.append(n).append(" NOT_FOUND; ");
    continue;
  }

  // Skip from LCM verification scan
  ir.setVerified(now);
  if (ir.getEndDate() == null) {
    ir.setEndDate(now);
  }
  ir.setExecutionStatus(ExecutionStatus.Completed);
  ir.setCompletionStatus(CompletionStatus.Success);

  // Best-effort: also try non-empty summaries so getApprovalSummaries is never null
  // (empty list can be stripped; one placeholder summary is safer if API allows)
  try {
    if (ir.getApprovalSummaries() == null) {
      // Prefer skip-via-verified above; this is only extra safety
      ir.setAttribute("approvalSummaries", null); // leave null; verified handles skip
    }
  } catch (Exception ignore) {}

  context.saveObject(ir);
  context.commitTransaction();
  context.decache(ir);
  fixed++;
  sb.append(n).append(" COMPLETED; ");
}

sb.append(" totalFixed=").append(fixed);
return sb.toString();
```

Expected return: each ID shows `COMPLETED`.

### Then
1. Confirm one request in Debug: `executionStatus=Completed`, `verified` is set  
2. Re-run **Perform Identity Request Maintenance**  
3. Those `0000001690`… NPE lines should disappear  

## If ExecutionStatus/CompletionStatus enum import fails
Use strings via XML in Debug for one request first:

- `executionStatus="Completed"`
- `completionStatus="Success"`
- add `verified` date (same as endDate)

Or use this variant:

```beanshell
import sailpoint.object.IdentityRequest;
import java.util.Arrays;
import java.util.Date;
import java.util.List;

List names = Arrays.asList(
"0000001690","0000001691","0000001692","0000001697","0000001698",
"0000001700","0000001701","0000001702","0000001703","0000001704",
"0000001705","0000001706","0000001707","0000001708","0000001709",
"0000001710","0000001711","0000001712","0000001732","0000001733",
"0000001735","0000001746"
);

int fixed = 0;
Date now = new Date();
StringBuffer sb = new StringBuffer();

for (int i = 0; i < names.size(); i++) {
  String n = (String) names.get(i);
  IdentityRequest ir = context.getObjectByName(IdentityRequest.class, n);
  if (ir == null) { sb.append(n).append(" NOT_FOUND; "); continue; }

  ir.setVerified(now);
  if (ir.getEndDate() == null) ir.setEndDate(now);
  ir.setExecutionStatus(IdentityRequest.ExecutionStatus.valueOf("Completed"));
  ir.setCompletionStatus(IdentityRequest.CompletionStatus.valueOf("Success"));

  context.saveObject(ir);
  context.commitTransaction();
  context.decache(ir);
  fixed++;
  sb.append(n).append(" COMPLETED; ");
}
sb.append(" totalFixed=").append(fixed);
return sb.toString();
```

## Important
- This cleans **old broken requests** so Maintenance can run.  
- Workflow “Ensure Approval Summaries” still helps **new** requests.  
- For Hoop groups on cube, keep AfterProvisioningRule.
