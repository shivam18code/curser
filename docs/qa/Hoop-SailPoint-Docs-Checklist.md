# What SailPoint docs say vs Hoop setup

Sources checked:
- [IdentityIQ Help home](https://documentation.sailpoint.com/identityiq/help/)
- [Web Services – Get Object](https://documentation.sailpoint.com/connectors/webservices/help/integrating_webservices/json_get_object.html)
- [Web Services – Configuration Parameters](https://documentation.sailpoint.com/connectors/webservices/help/integrating_webservices/additional_configuration_parameters.html)
- [Web Services – Account/Get Object multi-endpoint](https://documentation.sailpoint.com/connectors/webservices/help/integrating_webservices/xml_aggregation.html)
- [Web Services – Before Operation Rule](https://documentation.sailpoint.com/connectors/webservices/help/integrating_webservices/beforeoperationrule.html)

---

## 1) Get Object identity token (official)

Docs say use:

```text
$getObject.nativeIdentity$
```

([Get Object doc](https://documentation.sailpoint.com/connectors/webservices/help/integrating_webservices/json_get_object.html))

Your latest Hoop app has this (good). Bitbucket uses `$getobject...` (lowercase) — both are commonly accepted; match what works in your IIQ version.

**Missing earlier:** using `$plan.nativeIdentity$` on Get Object (not what docs show for getObject).

---

## 2) `isGetObjectRequiredForPTA` — not what you think

Official meaning: **Pass Through Authentication**.

When `true`, Get Object runs to verify the username exists during **login/PTA**, not to refresh the account after Add/Remove entitlement.

([Configuration Parameters](https://documentation.sailpoint.com/connectors/webservices/help/integrating_webservices/additional_configuration_parameters.html))

**You are not missing a post-provision “PTA getObject” feature here.** That flag does not fix stale groups after Modify.

---

## 3) Docs imply getObject after **Create**, not after every Add/Remove

Parameter `skipGetObjectInCreate`:

> skip the getObject call if it is present during the **Create** provisioning operation.

That means Create can trigger getObject by default. Docs do **not** document an equivalent “always getObject after Add Entitlement” switch for IIQ Web Services.

So expecting Add/Remove alone (with empty response mapping) to refresh the cube is **not** something the docs promise.

---

## 4) Multi-endpoint Get Object (Bitbucket pattern) — documented

Docs support parent → child endpoints so the second call can load extra attributes (e.g. groups) using `$response.<attr>$`.

([Account, Entitlement, or Get Object Aggregation](https://documentation.sailpoint.com/connectors/webservices/help/integrating_webservices/xml_aggregation.html))

Bitbucket does this (members, then groups).  
Hoop can stay **one** Get Object endpoint because `/api/users/{email}` already returns `groups` — as long as response mapping includes `groups` (yours does).

---

## 5) Empty Add/Remove response mapping

Docs emphasize response mapping for aggregation / Get Object. They do **not** require Add/Remove to map a full account response.

Empty Add/Remove mapping (like Bitbucket) is OK if the API only returns success. Then IIQ will not learn new `groups` from that HTTP response.

---

## 6) Before Operation Rule (docs)

Before Operation can rewrite URL/headers/body and return `requestEndPoint`.

([Before Operation Rule](https://documentation.sailpoint.com/connectors/webservices/help/integrating_webservices/beforeoperationrule.html))

Bitbucket’s Before Rule (token + URL swap) only makes admin APIs work. It does **not** refresh Link groups.  
Hoop’s Before Rule (merge groups/status) is for outbound PUT body — also not Link refresh.

---

## What you were missing (summary)

| Topic | Docs say | Your Hoop gap |
|-------|----------|----------------|
| Get Object URL | `$getObject.nativeIdentity$` | Was `$plan...`; now fixed |
| `isGetObjectRequiredForPTA` | Pass-through auth only | Not a post-Add/Remove refresh |
| Auto getObject after Create | Exists (`skipGetObjectInCreate` to skip) | N/A to Add/Remove refresh |
| Auto getObject after Add/Remove | Not documented as default for IIQ WS | Why cube stays stale |
| Groups on Get Object | Response mapping must include them | Already mapped |
| Bitbucket-style refresh | Working Get Object + AfterProvisioning (app-level) | Need AfterProv refresh / verify logs |

---

## 7) IIQ community: Get Object does **not** update Link by itself

Relevant IIQ thread (8.3):  
[WebServices connector GetObject operation does not update Link](https://developer.sailpoint.com/discuss/t/webservices-connector-getobject-operation-does-not-update-link/17108)

SailPoint-side explanation there:

- Get Object retrieves a ResourceObject for verification / further processing
- **Get Object alone does not update Link / identity attributes**
- To update the account on the identity (what you see on the cube), process via **Aggregator** (full or single-account aggregation UI/task)

That matches your Hoop symptom: Add/Remove succeeds on target, but cube groups stay old until you manually aggregate.

So even with a correct Get Object operation in the Application XML, **IIQ will not necessarily write new `groups` onto the Link after Modify**. AfterProvisioning (or an aggregation) is still required to update what you see.

---

## PDF note

Local path `c:\Users\shivam.gore\Downloads\8.2 SailPoint Web Services Connector Guide.pdf` is not readable from this cloud agent. Upload/attach that PDF into the workspace or chat to search it line-by-line.

