# Collibra-Propose-Business-Term-Select-Domain
Business users use this workflow to propose a new **Business Term** or **Acronym**. The user picks which **Glossary domain** gets the new asset, fills in a short form, and the asset is created right away.

# Propose New Business Term – Collibra Workflow

| | |
|---|---|
| **Process ID** | `intakeBusinessTerm` |
| **Process name** | Propose New Business Term |
| **File** | `intakeBusinessTerm_glossaryPicker.bpmn` |
| **Start type** | Global (not tied to a specific asset, domain or community) |
| **Tested as** | System Administrator only. See [Testing with non-admin users](#6-testing-with-non-admin-users) |

---

## 1. What it does

Business users use this workflow to propose a new **Business Term** or **Acronym**. The user picks which **Glossary domain** gets the new asset, fills in a short form, and the asset is created right away.

> **No approval step.** Submitting the form creates the asset directly. Its status is the default for the asset type (normally *Candidate*). An approval step can be added later.

---

## 2. Process flow

```
 (Start) ──► [Get Glossary Domains] ──► [Propose Business Term] ──► [Create Asset] ──► (End)
              script task               user task (initiator)       script task
```

| Step | ID | Type | What happens |
|---|---|---|---|
| Start | `startevent1` | Start event | No form. Records the initiator in `${initiator}`. |
| Get Glossary Domains | `getGlossaryDomains` | Groovy script | Looks up every domain of type **Glossary** (`…010001`) and stores their IDs, comma-separated, in `glossaryDomainIds`. Uses a fallback domain if none are found. Writes `[intakeBusinessTerm]` messages to the log. |
| Propose Business Term | `proposeTerm` | User task | The proposal form. Assigned to the user who started the workflow: `user(${initiator})`. |
| Create Asset | `createAsset` | Groovy script | Creates the asset in the chosen Glossary domain, adds the Definition and Note attributes, and adds "uses" relations to the related assets. Stores the new asset ID in `outputCreatedTermId`. |
| End | `endevent1` | End event | Done. |

**Why is the form a user task rather than a start form?** Collibra's `domain` form field can't filter by domain type on its own. The list of Glossary domains has to be built at runtime, before the form is shown. The start event can't run code before its form appears, so the form is the first user task instead. See [Known limitations](#9-known-limitations).

---

## 3. Form fields (user task `proposeTerm`)

| Field ID | Label | Type | Required | Notes |
|---|---|---|---|---|
| `intakeVocabulary` | Glossary | `domain` | Yes | Drop-down limited to `${glossaryDomainIds}` (`proposedFixed=true`, single value). |
| `signifier` | Name | `string` | Yes | Asset name and display name. |
| `conceptType` | Type | `assetType` | Yes | Limited to Business Term (`…011001`) and Acronym (`…011003`). |
| `definition` | Proposed Definition | `textarea` | No | Saved as the Definition attribute. |
| `usesrelation` | Related Assets | `term` | No | Multiple values, filtered to asset type `…031000`. |
| `note` | Reason for proposal | `textarea` | No | Saved as the Note attribute. |
| `submit` | Propose | `button` | – | Submits the form. |

Hidden configuration fields (`readable="false"`) are covered in the next section.

---

## 4. Configuration options

All IDs are Collibra out-of-the-box UUIDs unless you have customised your operating model. **Check each one in your environment before deploying** (Settings > Operating Model).

### 4.1 In the `getGlossaryDomains` script

| Variable | Default | Purpose |
|---|---|---|
| `glossaryDomainTypeUuid` | `00000000-0000-0000-0000-000000010001` | Domain type shown in the drop-down (Glossary). Change it to offer another domain type. |
| `fallbackDomainUuid` | `00000000-0000-0000-0000-000000006013` | Used if no Glossary domains are found. **Make sure this domain exists in your environment**, or point it at a real intake domain. |
| `limit` | `500` | Page size for the domain lookup. It keeps fetching pages until all domains are found. |

**Optional: limit the list to one community.** Add `.communityId(string2Uuid("<community-uuid>"))` to the `FindDomainsRequest.builder()` chain. Check that the builder method exists in your Collibra version first (see [Troubleshooting](#8-troubleshooting)).

### 4.2 Hidden fields on the user task

| Field ID | Default | Purpose |
|---|---|---|
| `usesRelationTypeUuid` | `00000000-0000-0000-0000-000000007004` | Relation type used for "Related Assets". |
| `definitionAttributeTypeUuid` | `00000000-0000-0000-0000-000000000202` | Definition attribute type. |
| `noteAttributeTypeUuid` | `00000000-0000-0000-0000-000000003116` | Note attribute type. |

### 4.3 Other form filters

| Setting | Location | Default |
|---|---|---|
| Asset types the user can choose | `conceptType` > `proposedValues` | Business Term, Acronym |
| Asset type allowed as a related asset | `usesrelation` > `conceptType` | `00000000-0000-0000-0000-000000031000` |

> **Check the relation type fits.** Collibra rejects a relation whose source or target asset type isn't allowed by that relation type. Make sure relation `…7004` accepts Business Term and Acronym as the source and `…031000` as the target. If not, change the relation type or the `usesrelation` filter.

---

## 5. Deployment steps

1. **Upload the file**
   - Go to **Settings > Workflows > Definitions** and click **Upload a file**.
   - Select `intakeBusinessTerm_glossaryPicker.bpmn`.
   - If an older version with the same process ID (`intakeBusinessTerm`) exists, it's replaced. Running instances keep using the old version.

2. **Configure the definition.** Open **Propose New Business Term** and set:

   | Setting | Recommended value | Why |
   |---|---|---|
   | **Enabled** | On | |
   | **Applies to** | *Global* | Lets the workflow start from the global Create / Workflows menu without a business item. |
   | **Start events** | *User* | Users start it by hand. |
   | **Start roles** (who can start it) | The role(s) for your proposers, e.g. *Everyone*, or a custom *Glossary Contributor* role | **This is the most common reason it works for admins and not for others.** |
   | **Stop roles** | Admin / Sysadmin | Who can cancel running instances. |
   | **Show in global create menu** (if available in your version) | On | So users can find it easily. |
   | **Guest user can start** | Off | Unless you want anonymous proposals. |
   | **Only one instance per business item** | Off | Not relevant for a global workflow. |

3. **Check the operating model IDs.** Confirm every UUID in [Section 4](#4-configuration-options) exists, especially `fallbackDomainUuid`.

4. **Check that Glossary domains exist.** There must be at least one domain of type Glossary, or the drop-down shows only the fallback domain.

5. **Smoke test as a sysadmin.** Start the workflow and confirm the drop-down lists the Glossary domains. Submit, then check the new asset, its attributes, and its relations.

6. **Test as non-admins.** Follow the next section.

---

## 6. Testing with non-admin users

This workflow has **only been tested as a System Administrator**, who bypasses most permission checks. Before rolling it out, test with accounts that match your real proposers.

### 6.1 Suggested test users

| Persona | License | Global role | Resource role on target Glossary |
|---|---|---|---|
| A. Typical proposer | Consumer / Viewer | Standard user role | None |
| B. Contributor | Author / Creator | Standard user role | None |
| C. Steward | Author / Creator | Standard user role | Steward (or equivalent) |

### 6.2 What to check for each persona

| # | Check | Expected result | If it fails |
|---|---|---|---|
| 1 | Workflow appears in the global menu | Visible | Add the user's role to **Start roles**. Check that the definition is enabled and set to *Global*. |
| 2 | Workflow starts without "Unexpected error" | Form task opens or appears in **Tasks** | Check the log (Section 8). |
| 3 | Form opens for the initiator | Task assigned to that user | Confirm `flowable:candidateUsers="user(${initiator})"` is accepted. Check the user's license allows completing tasks. |
| 4 | Glossary drop-down is filled in | Lists all Glossary domains | See Section 8, "Drop-down is empty". |
| 5 | Asset is created on submit | Asset exists in the chosen domain | Check whether the script task ran with the user's own permissions (see below). |
| 6 | Related Assets picker finds assets | Assets of type `…031000` are searchable | The user may lack view rights on those assets. |

### 6.3 Permissions to confirm

- **Starting the workflow:** controlled by **Start roles** on the definition, and by the user's license. Check which license types can start workflows and complete tasks in your version.
- **Seeing domains in the drop-down:** Collibra's `domain` field lists domains even when the user can't view them. That's intended, but users may see Glossaries they can't browse.
- **Creating the asset:** confirm whether a user with **no** create rights in the target domain (Persona A) can still create assets through the workflow. If creation fails for Persona A but works for Persona C, the script is running with the user's own permissions. In that case either:
  - give proposers a resource role with *Asset > Add* on the Glossary domains or their community, or
  - add a Collibra-supported way to run the script task as a service account.
- **Related Assets relations:** adding relations may need edit rights on the target assets, just like creating the asset.

---

## 7. Process variables

| Variable | Set by | Type | Description |
|---|---|---|---|
| `initiator` | Start event | String (user ID) | User who started the workflow. |
| `glossaryDomainIds` | `getGlossaryDomains` | String | Comma-separated Glossary domain UUIDs. |
| `intakeVocabulary` | Form | UUID | Selected Glossary domain. |
| `signifier` | Form | String | Asset name. |
| `conceptType` | Form | UUID | Selected asset type. |
| `definition` | Form | String | Proposed definition. |
| `note` | Form | String | Reason for proposal. |
| `usesrelation` | Form | List of UUIDs | Related assets. |
| `outputCreatedTermId` | `createAsset` | String | UUID of the new asset. |

---

## 8. Troubleshooting

### "Unexpected error – The 'Propose New Business Term' workflow was unable to start"

Everything up to the first user task (**Get Glossary Domains**) runs as soon as the workflow starts. If that script throws an error, the start is rolled back and you get this generic message.

1. **Read the log.** In **Collibra Console > your environment > Data Governance Center > Logs**, download `dgc.log`. Search for `[intakeBusinessTerm]`, `intakeBusinessTerm` or `Propose New Business Term`.
2. **Common causes:**
   - `MissingMethodException` on a `FindDomainsRequest` builder method: that method doesn't exist in your API version. Remove it. (`includeSubTypes` and `excludeMeta` were removed for this reason.)
   - `No such property`: a variable name is misspelled, or a variable is missing.
   - Invalid UUID: a configured ID has stray spaces or doesn't exist.
3. **Isolate the problem.** Temporarily replace the whole lookup script with:
   ```groovy
   execution.setVariable("glossaryDomainIds", "00000000-0000-0000-0000-000000006013")
   ```
   If the workflow now starts, the problem is in the domain lookup. If it still fails, check the definition settings (Section 5).

### The Glossary drop-down is empty or shows every domain

- Check the log for `Found N Glossary domain(s)`. If N is 0, no domains of type `…010001` exist. Check the type ID.
- If N > 0 but the drop-down is still wrong, your Collibra version may not fill in the `${glossaryDomainIds}` expression inside `proposedValues`. Fallback: list the domain IDs directly, as a static string, in `proposedValues`.

### The asset isn't created, or the task fails on submit

- Check the log for errors from `assetApi.addAsset`. Common causes:
  - an asset with the same name already exists in that domain;
  - the asset type isn't allowed in that domain type (check the assignment for Glossary + Business Term / Acronym);
  - missing permissions (Section 6.3).
- If relation creation fails, check the relation type rules in Section 4.3.

---

## 9. Known limitations

- **Two-step experience:** the user starts the workflow, then fills in the form as a task. Depending on your version and settings, the task may open right away or appear under **Tasks**.
- **No approval or review step.** Assets are created as soon as the form is submitted.
- **No duplicate-name check** before creation. Collibra's own uniqueness rules apply.
- **No notification** to stewards or the requester after creation.
- The domain list is taken **at start time**. A Glossary created while an instance is waiting at the form won't appear until a new instance is started.

---

## 10. Possible enhancements

- An approval step routed to the chosen Glossary's Steward or Business Steward.
- An email or in-app notification to the requester with a link to the new asset (`outputCreatedTermId`).
- A duplicate-name check against the chosen domain before creating the asset.
- Limiting the drop-down to Glossaries in a specific community, or to those the user is responsible for.

---

## 11. Change log

| Date | Change |
|---|---|
| 2026-09-29 | Added Glossary domain drop-down (`getGlossaryDomains` script + `proposeTerm` user task). Fixed namespace quoting and the trailing space in the Note attribute type UUID. |
| 2026-09-29 | Removed `includeSubTypes` / `excludeMeta` from the domain lookup. Added `[intakeBusinessTerm]` logging to fix the "unable to start" error. |
