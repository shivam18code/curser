# Hoop Application — Manual Test Cases (SailPoint IIQ)

**Application:** Hoop (`Web Services` v2)  
**Environment:** STG — `https://hoopui-stg.pfa.amlakint.com`  
**Identity attribute:** `email`  
**Entitlement attribute:** `groups` (multi)  
**BeforeOperation rule:** `Amlak-Rule-BeforeRule-Hoop`  
**AfterOperation rule:** `Amlak-Rule-AfterRule-Hoop` (Group Aggregation)  
**Correlation:** `Amlak-Correlation-Hoop`  
**Logger:** `com.amlak.application.hoop`

Use this document for manual QA of aggregation and provisioning against Hoop. Mark **Pass / Fail / Blocked** and record evidence (IIQ request ID, Hoop user state, log snippets).

---

## 0. Prerequisites

| # | Check | Expected |
|---|--------|----------|
| P-01 | Application `Hoop` exists in IIQ | Type Web Services, base URL STG |
| P-02 | OAuth credentials valid | Test Connection succeeds |
| P-03 | Rule `Amlak-Rule-BeforeRule-Hoop` imported and attached | Add / Remove / Disable / Enable endpoints |
| P-04 | `throwProvBeforeRuleException = true` | BeforeRule failures abort provisioning |
| P-05 | `fixedPlanMultivaluedAttribute = true` | `$plan.groups$` expands as JSON array |
| P-06 | `isGetObjectRequiredForPTA = true` | Get Object runs for plan evaluation |
| P-07 | Logger `com.amlak.application.hoop` at INFO/DEBUG | BeforeRule ENTRY/EXIT/finalBody visible |
| P-08 | Test identities prepared | See test data below |
| P-09 | Hoop UI/API access for verification | Can GET `/api/users/{email}` and view groups |

### Recommended test data

| Alias | Purpose | Starting Hoop state |
|-------|---------|---------------------|
| **U-ACTIVE** | Happy-path entitlement & disable | `status=active`, groups e.g. `["engineering"]` |
| **U-INACTIVE** | Status preservation on entitlement | `status=inactive`, groups e.g. `["engineering"]` |
| **U-MULTI** | Remove one of many groups | `status=active`, groups `["engineering","ops","finance"]` |
| **U-NEW** | Create Account | No Hoop account yet (email = IIQ identity email) |
| **G-A / G-B** | Entitlement values | Valid group names from Group Aggregation |

### How to verify results

1. **IIQ:** Manage Accounts / Access Request / Provisioning Transaction / Identity Cube → Accounts → Hoop  
2. **Hoop API:** `GET {baseUrl}/api/users/{email}` → check `email`, `name`, `status`, `groups`  
3. **Logs:** search `[Amlak-Rule-BeforeRule-Hoop]` / `[Hoop-BeforeRule]` for `ENTRY`, `requestedGroups`, `Preserving existing status` / `Setting status`, `ADD`/`REMOVE`, `finalBody`, `EXIT`

### Expected rewritten PUT body (After BeforeRule)

```json
{
  "email": "<nativeIdentity from URL>",
  "name": "<from GET, else email>",
  "status": "active|inactive",
  "groups": ["..."]
}
```

| Operation | Status in final body | Groups in final body |
|-----------|----------------------|----------------------|
| Add Entitlement | **Preserve** current | current ∪ requested |
| Add Entitlement + user **404** | `active` (bootstrap) | requested only; switches to **POST** `/api/users` |
| Remove Entitlement | **Preserve** current | current − requested |
| Disable Account | `inactive` | **Preserve** current |
| Enable Account | `active` | **Preserve** current |

> Template static `"status":"active"` on Add/Remove must be **ignored** by the BeforeRule.  
> If IIQ runs **Modify / Add Entitlement** but Hoop has no user (`GET 404`), the corrected BeforeRule bootstraps **Create**. Remove / Enable / Disable on missing user still fail closed.

---

## 1. Connectivity & aggregation

### TC-CONN-01 — Test Connection

| Field | Value |
|-------|-------|
| **Priority** | P0 |
| **Endpoint** | Test Connection → `GET /api/users` |
| **Steps** | 1. Open Application → Configuration → Test Connection. 2. Run test. |
| **Expected** | Success. No auth/timeout errors. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

### TC-AGG-01 — Account Aggregation (full)

| Field | Value |
|-------|-------|
| **Priority** | P0 |
| **Endpoint** | Account Aggregation → `GET /api/users` |
| **Steps** | 1. Run Account Aggregation for Hoop. 2. Open a known user cube. 3. Confirm Hoop link and attributes. |
| **Expected** | Accounts created/updated. Attributes mapped: `id`, `name`, `email`, `status`, `role`, `groups`. Correlation via `Amlak-Correlation-Hoop` works for matching identities. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

### TC-AGG-02 — Group Aggregation

| Field | Value |
|-------|-------|
| **Priority** | P0 |
| **Endpoint** | Group Aggregation → `GET /api/users/groups` + AfterRule `Amlak-Rule-AfterRule-Hoop` |
| **Steps** | 1. Run Group Aggregation. 2. Open Application → Entitlements / Groups. |
| **Expected** | Groups appear as entitlements. Attribute `group` populated. AfterRule does not drop/duplicate valid groups. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

### TC-AGG-03 — Single Account Aggregation (Get Object)

| Field | Value |
|-------|-------|
| **Priority** | P0 |
| **Endpoint** | Get Object → `GET /api/users/$plan.nativeIdentity$` |
| **Steps** | 1. From Identity Cube → Accounts → Hoop → refresh / Get Object (or trigger PTA path that requires Get Object). 2. Compare IIQ account attrs to Hoop API. |
| **Expected** | `email`, `status`, `groups`, `id`, `role` refresh correctly. |
| **Known issue** | Response mapping key is `"name "` (trailing space). **`name` may not map** until fixed to `"name"`. Document actual behavior. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

## 2. Create Account

### TC-CREATE-01 — Create new Hoop account (happy path)

| Field | Value |
|-------|-------|
| **Priority** | P0 |
| **Endpoint** | Create Account → `POST /api/users` (no BeforeRule) |
| **Precondition** | Identity **U-NEW** has email, first/last name; no Hoop account |
| **Steps** | 1. Request new Hoop account (Manage Accounts / LCM). 2. Confirm Create form shows `displayName` = first+last, `email` = identity email. 3. Submit and wait for provisioning success. 4. Verify via Hoop GET and IIQ cube. |
| **Expected** | Body sent: `{"email":"<email>","name":"<displayName>","status":"active"}`. Account exists in Hoop with `status=active`. Linked on cube. Native identity = email. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

### TC-CREATE-02 — Create Account when user already exists

| Field | Value |
|-------|-------|
| **Priority** | P1 |
| **Precondition** | Hoop already has **U-ACTIVE** |
| **Steps** | Attempt Create Account for same email. |
| **Expected** | Provisioning fails with clear API/IIQ error; no duplicate/corrupt account. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

### TC-CREATE-03 — Create Account does not assign groups

| Field | Value |
|-------|-------|
| **Priority** | P1 |
| **Precondition** | `createAccountWithEntReq` is false |
| **Steps** | Create account only (no entitlement in same request if UI allows). |
| **Expected** | User created without forced groups (or empty groups per Hoop default). Entitlements require separate Add Entitlement. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

## 3. Add Entitlement

### TC-ADD-01 — Add group to active user (merge)

| Field | Value |
|-------|-------|
| **Priority** | P0 |
| **Endpoint** | Add Entitlement (`uniqueNameForEndPoint`: **Add Entitilement**) → PUT + BeforeRule |
| **Precondition** | **U-ACTIVE**: `status=active`, groups include `G-A` only |
| **Steps** | 1. Request Add Entitlement `G-B` for **U-ACTIVE**. 2. Check IIQ provisioning success. 3. GET Hoop user. 4. Check logs for ADD + `Preserving existing status`. |
| **Expected** | Groups = `G-A` ∪ `G-B` (both present). Status remains `active`. `finalBody` includes email, name, status, full groups array. Existing group not wiped. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

### TC-ADD-02 — Add group to inactive user (must NOT re-enable)

| Field | Value |
|-------|-------|
| **Priority** | P0 — **regression** |
| **Precondition** | **U-INACTIVE**: `status=inactive`, has `G-A` |
| **Steps** | Add entitlement `G-B`. Verify Hoop GET + logs. |
| **Expected** | Groups include `G-A` and `G-B`. **Status stays `inactive`**. Log shows `Preserving existing status=inactive`. Must **not** apply template `"status":"active"`. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

### TC-ADD-03 — Add already-assigned group (idempotent)

| Field | Value |
|-------|-------|
| **Priority** | P2 |
| **Precondition** | User already has `G-A` |
| **Steps** | Add `G-A` again. |
| **Expected** | Still one `G-A` (set semantics). Status unchanged. Provisioning succeeds or no-ops cleanly. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

### TC-ADD-04 — Add multiple groups in one request (if UI/plan allows)

| Field | Value |
|-------|-------|
| **Priority** | P1 |
| **Steps** | Request multiple groups in one Add Entitlement plan. |
| **Expected** | All requested groups merged into current set. `requestedGroups` in logs shows full list / JSON array. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

### TC-ADD-05 — Add Entitlement when Hoop user does not exist (404 bootstrap Create)

| Field | Value |
|-------|-------|
| **Priority** | P0 — **regression for Modify + missing target** |
| **Precondition** | IIQ has (or plans) a Hoop account link / Modify Add Entitlement, but `GET /api/users/{email}` returns **404** `user ... not found`. Native identity must be a real email (not `???`). |
| **Steps** | 1. Request Add Entitlement for a user missing in Hoop. 2. Check logs for `bootstrapping Create` and `Switched endpoint to Create POST`. 3. Verify Hoop user exists with requested groups and `status=active`. |
| **Expected** | Provisioning **succeeds**. BeforeRule does **not** throw on 404. User created via `POST /api/users` with `email`, `name`, `status=active`, `groups` = requested. |
| **Fail if** | Still see `GET failed status=404` RuntimeException (old rule). Or nativeIdentity is `???` / invalid — fix IIQ link email first. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

## 4. Remove Entitlement

### TC-REM-01 — Remove one group from multi-group user

| Field | Value |
|-------|-------|
| **Priority** | P0 |
| **Endpoint** | Remove Entitlement → PUT + BeforeRule |
| **Precondition** | **U-MULTI**: groups `["engineering","ops","finance"]`, status `active` |
| **Steps** | Remove `ops` only. Verify Hoop + logs (`REMOVE groups`). |
| **Expected** | Remaining groups `engineering`, `finance`. Status still `active`. Other groups not deleted. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

### TC-REM-02 — Remove group from inactive user (must NOT re-enable)

| Field | Value |
|-------|-------|
| **Priority** | P0 — **regression** |
| **Precondition** | Inactive user with `G-A` and `G-B` |
| **Steps** | Remove `G-B`. |
| **Expected** | `G-B` gone; `G-A` remains; **status still `inactive`**. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

### TC-REM-03 — Remove last remaining group

| Field | Value |
|-------|-------|
| **Priority** | P1 |
| **Precondition** | User has only `G-A` |
| **Steps** | Remove `G-A`. |
| **Expected** | `groups` empty `[]` (or Hoop-equivalent empty). Account still exists; status unchanged. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

### TC-REM-04 — Remove group user does not have

| Field | Value |
|-------|-------|
| **Priority** | P2 |
| **Steps** | Remove a group not on the account. |
| **Expected** | Groups unchanged (minus nothing). No wipe. Status unchanged. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

## 5. Disable / Enable Account

### TC-DIS-01 — Disable active account

| Field | Value |
|-------|-------|
| **Priority** | P0 |
| **Endpoint** | Disable Account → PUT + BeforeRule (empty template body) |
| **Precondition** | **U-ACTIVE** with one or more groups |
| **Steps** | Disable account from IIQ. Check Hoop GET + logs (`Setting status from enable/disable=inactive`). |
| **Expected** | `status=inactive`. **Groups unchanged**. Name/email preserved. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

### TC-EN-01 — Enable inactive account

| Field | Value |
|-------|-------|
| **Priority** | P0 |
| **Endpoint** | Enable Account → PUT + BeforeRule |
| **Precondition** | Account already disabled (from TC-DIS-01 or **U-INACTIVE**) with groups |
| **Steps** | Enable account. Verify Hoop + logs (`status=active`, groups preserved). |
| **Expected** | `status=active`. Groups identical to before enable. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

### TC-DIS-02 — Disable then Add Entitlement (combo)

| Field | Value |
|-------|-------|
| **Priority** | P0 — **regression** |
| **Steps** | 1. Disable user. 2. Add a new group. 3. Confirm still inactive with merged groups. |
| **Expected** | Status remains inactive after Add. Groups include new entitlement. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

## 6. BeforeRule failure / fail-closed

### TC-RULE-01 — GET 404 on Remove / Enable / Disable still aborts

| Field | Value |
|-------|-------|
| **Priority** | P0 — **safety** |
| **Precondition** | Target email does not exist in Hoop (404). `throwProvBeforeRuleException=true` |
| **Steps** | Trigger **Remove** or **Disable** or **Enable** (not Add). Watch IIQ provisioning + logs. |
| **Expected** | BeforeRule throws with clear “does not exist / Create the account first”; provisioning **fails**. No wipe. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

### TC-RULE-01b — GET non-404 error aborts even on Add

| Field | Value |
|-------|-------|
| **Priority** | P0 — **safety** |
| **Precondition** | Force GET failure other than 404 (e.g. 401/500) |
| **Steps** | Trigger Add Entitlement. |
| **Expected** | BeforeRule throws; provisioning fails; no bootstrap Create. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

### TC-RULE-02 — Identity mismatch aborts

| Field | Value |
|-------|-------|
| **Priority** | P1 |
| **Steps** | If reproducible: URL identity vs GET returned `email` differ. |
| **Expected** | RuntimeException; no PUT applied. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

### TC-RULE-03 — Logs show rewritten body before PUT

| Field | Value |
|-------|-------|
| **Priority** | P1 |
| **Steps** | Run any Add/Remove/Enable/Disable. Capture `finalBody=` and `EXIT` lines. |
| **Expected** | `finalBody` JSON has `email`, `name`, `status`, `groups`. Matches post-provision Hoop GET. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

## 7. Correlation & forms

### TC-CORR-01 — Correlation after aggregation

| Field | Value |
|-------|-------|
| **Priority** | P1 |
| **Steps** | Aggregate accounts; open identity that should match on email. |
| **Expected** | Hoop account correlates via `Amlak-Correlation-Hoop` (email). Uncorrelated orphans only where email does not match. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

### TC-FORM-01 — Create form field population

| Field | Value |
|-------|-------|
| **Priority** | P2 |
| **Steps** | Open Create Account request form. |
| **Expected** | `displayName` = firstname + " " + lastname; `email` = identity email. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

### TC-FORM-02 — Update form exists but no Update Account endpoint

| Field | Value |
|-------|-------|
| **Priority** | P2 |
| **Steps** | Attempt attribute-only update (name/email) from IIQ if exposed. |
| **Expected** | Document actual behavior: Update form exists, but **no Update Account operation** is configured — update may be unsupported / fail / no-op. Capture result for backlog. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

## 8. Negative & edge cases

### TC-NEG-01 — Invalid / unknown group name

| Field | Value |
|-------|-------|
| **Priority** | P1 |
| **Steps** | Add entitlement value that does not exist in Hoop. |
| **Expected** | Clear failure from Hoop or IIQ; existing groups not wiped if BeforeRule GET succeeded then PUT rejected — confirm actual Hoop behavior. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

### TC-NEG-02 — Special characters in email (URL encoding)

| Field | Value |
|-------|-------|
| **Priority** | P2 |
| **Steps** | User email with `+` or encoded chars if present in STG. Trigger Get Object / Add. |
| **Expected** | BeforeRule URL-decodes identity; GET/PUT target correct user. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

### TC-NEG-03 — Password / Unlock / Authenticate features

| Field | Value |
|-------|-------|
| **Priority** | P3 |
| **Steps** | Confirm whether PASSWORD, UNLOCK, AUTHENTICATE appear in UI despite being in `featuresString`. |
| **Expected** | Document: no matching endpoints configured → features should not be usable (or are inert). No accidental broken ops. |
| **Result** | ☐ Pass ☐ Fail ☐ Blocked |
| **Notes** | |

---

## 9. Configuration defects to validate / fix during testing

| ID | Defect | Impact | Suggested fix | Verified? |
|----|--------|--------|---------------|-----------|
| DEF-01 | Get Object mapping key `"name "` (trailing space) | `name` may not populate on Single Account Aggregation / PTA | Change to `"name"` | ☐ |
| DEF-02 | Endpoint name typo `Add Entitilement` | Confusing; rule still treats as Add (not remove/enable/disable) | Rename to `Add Entitlement` (keep BeforeRule attached) | ☐ |
| DEF-03 | Static `"status":"active"` in Add/Remove templates | Dangerous if BeforeRule missing/old | Rely on corrected BeforeRule; optionally remove static status from template | ☐ |
| DEF-04 | No Update Account / Delete Account endpoints | Update form / delete not supported | Backlog if business needs them | ☐ |
| DEF-05 | Live OAuth token was pasted outside IIQ | Credential exposure risk | Rotate STG token if exposed; never commit secrets | ☐ |

---

## 10. Suggested execution order

1. P-01 … P-09 prerequisites  
2. TC-CONN-01  
3. TC-AGG-02 → TC-AGG-01 → TC-AGG-03  
4. TC-CREATE-01  
5. TC-ADD-01 → TC-REM-01 → TC-DIS-01 → TC-EN-01  
6. **Regression pack:** TC-ADD-02, TC-ADD-05, TC-REM-02, TC-DIS-02, TC-RULE-01, TC-RULE-01b  
7. Remaining P1/P2 cases  
8. Record DEF-01 … DEF-05 outcomes  

---

## 11. Test run summary

| Field | Value |
|-------|-------|
| **Tester** | |
| **Date** | |
| **IIQ env** | |
| **Hoop env** | STG `https://hoopui-stg.pfa.amlakint.com` |
| **BeforeRule version** | Amlak-Rule-BeforeRule-Hoop (import date: ____) |
| **Build / commit** | |
| **Overall result** | ☐ Pass ☐ Fail ☐ Pass with defects |
| **Defects opened** | |
| **Sign-off** | |

### Results tally

| Suite | Total | Pass | Fail | Blocked | N/A |
|-------|-------|------|------|---------|-----|
| Connectivity & aggregation | 4 | | | | |
| Create Account | 3 | | | | |
| Add Entitlement | 5 | | | | |
| Remove Entitlement | 4 | | | | |
| Disable / Enable | 3 | | | | |
| BeforeRule fail-closed | 4 | | | | |
| Correlation & forms | 3 | | | | |
| Negative / edge | 3 | | | | |
| **Total** | **29** | | | | |

---

## 12. Quick reference — endpoints

| # | Operation | Endpoint name | Method | URL | BeforeRule | AfterRule |
|---|-----------|---------------|--------|-----|------------|-----------|
| 1 | Test Connection | Test Connection | GET | `/api/users` | — | — |
| 2 | Account Aggregation | Account Aggregation | GET | `/api/users` | — | — |
| 3 | Group Aggregation | Group Aggregation | GET | `/api/users/groups` | — | Amlak-Rule-AfterRule-Hoop |
| 4 | Create Account | Create Account | POST | `/api/users` | — | — |
| 5 | Add Entitlement | **Add Entitilement** | PUT | `/api/users/$plan.nativeIdentity$` | BeforeRule | — |
| 6 | Remove Entitlement | Remove Entitlement | PUT | `/api/users/$plan.nativeIdentity$` | BeforeRule | — |
| 7 | Disable Account | Disable Account | PUT | `/api/users/$plan.nativeIdentity$` | BeforeRule | — |
| 8 | Enable Account | Enable Account | PUT | `/api/users/$plan.nativeIdentity$` | BeforeRule | — |
| 9 | Get Object | Single Account Aggregation | GET | `/api/users/$plan.nativeIdentity$` | — | — |

### Account schema

`id`, `name`, `email` (identity), `status`, `role`, `groups` (entitlement, multi → group)

### Group schema

`group` (identity + display)
