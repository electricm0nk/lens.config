# Adversarial Review: fourdogs-etailpet-api-sales / businessplan

**Reviewed:** 2026-04-27T00:00:00Z
**Source:** phase-complete
**Overall Rating:** pass-with-warnings

## Summary

The BusinessPlan artifacts (`prd.md`, `ux-design.md`) are substantive and well-grounded in prior spike
research. The spike-gate pattern correctly contains the highest-risk assumption (www credential
validity on the report route). Two High findings and four Medium findings remain. The most material
gaps are: (1) absence of a fallback plan if the spike fails, and (2) a fire-and-forget observability
gap between trigger success and email delivery. These do not block TechPlan from proceeding but
must be explicitly accepted or addressed. The data pipeline downstream of emailfetcher is
undocumented and has no owner — this is a business case gap that limits confidence in the stated
Kaylee SKU-velocity goal. TechPlan may proceed with these findings documented and accepted.

---

## Findings

### Critical

*None.*

---

### High

| # | Dimension | Finding | Recommendation |
|---|-----------|---------|----------------|
| H1 | Logic Flaws / Assumptions | The PRD has no documented fallback if `sales-0-01` fails. All Sprint 1–2 stories, the team's investment, and the entire delivery plan assume spike success. If the `www` auth fails on the report route, or the email never arrives, there is no documented alternative path. | Add a "Spike Failure Decision" section to `prd.md` or `tech-plan.md`. At minimum: state whether the feature is abandoned, re-scoped, or blocked on an eTailPet support ticket. |
| H2 | Coverage Gaps / Complexity | Fire-and-forget: `trigger_success` (HTTP 200 from `www`) does not mean an email was delivered. eTailPet's outbound email infrastructure is outside the operator's control. The PRD's success metric "emailfetcher inbox delivery within 10 minutes" has no monitoring mechanism. Silent email delivery failures could produce a week-long data gap with no alert. | Accept this as a deferred monitoring story (e.g., periodic emailfetcher delivery check) or add an explicit Accepted Risk noting that email non-delivery is undetected in v1. |

---

### Medium

| # | Dimension | Finding | Recommendation |
|---|-----------|---------|----------------|
| M1 | Cross-Feature Dependencies | emailfetcher routing rule for sales-line emails is unconfirmed. The PRD says emailfetcher is out-of-scope, but a routing rule for the (yet-unknown) sales-line email subject must exist for any data to flow. | Confirm in `sales-0-01` that the email subject is known; add an emailfetcher routing check to the staging validation story (`sales-2-03`). Flag if emailfetcher changes are required. |
| M2 | Coverage Gaps | The end-to-end data pipeline (trigger → email → emailfetcher → parse xlsx → store → Kaylee) is documented only for the first step. No feature, owner, or timeline exists for downstream stages. The PRD's Kaylee SKU-velocity justification is unsupported without a follow-on plan. | Add a "Downstream Pipeline" note to `prd.md` naming the next stage and identifying whether it is an existing feature or a gap. |
| M3 | Coverage Gaps | `TRIGGER_LOOKBACK_DAYS=7` runs daily, potentially re-requesting overlapping data windows. eTailPet deduplication behavior and emailfetcher's response to duplicate report emails are unacknowledged. | Add a note to `prd.md` or `tech-plan.md`: confirm in the spike whether eTailPet deduplicates, or accept that downstream emailfetcher must handle duplicate reports. |
| M4 | Logic Flaws | `prd.md` frontmatter `key_decisions` states "Same Vault credentials reused from transactions" but `open_questions` asks to "Confirm www legacy auth accepts existing Vault credentials on the report route." These contradict: the first is a confirmed decision, the second is an unconfirmed assumption. | Reclassify the credential sharing as an `assumption` (not `key_decision`) until the spike confirms it. |

---

### Low

| # | Dimension | Finding | Recommendation |
|---|-----------|---------|----------------|
| L1 | Coverage Gaps | Date format `YYYY-MM-DD` (vs. pos `MM/DD/YYYY`) is documented only in `tech-plan.md`, not in the PRD. This is a cross-feature behavioral difference that tech reviewers may miss in the PRD. | Add a sentence under FR-1.5 or Constraints in `prd.md` noting the www date format differs from pos. |
| L2 | Assumptions | TO_EMAIL parameter: the PRD and ux-design don't confirm whether the www trigger accepts a `to_email` parameter or uses a fixed retailer-configured inbox. If a parameter is required, it becomes a required env var. | Add to `open_questions` in `prd.md`: "Does the www trigger require TO_EMAIL parameter, or does it use a retailer-configured recipient?" |
| L3 | Complexity | Staging / production isolation: if staging and production share the same eTailPet retailer credentials and inbox, staging triggers send real emails to the real inbox with no isolation boundary. | Confirm whether a staging-isolated eTailPet schema or inbox exists; add a note to the runbook in `ux-design.md`. |

---

## Accepted Risks

| # | Finding | Accepted By | Rationale |
|---|---------|-------------|-----------|
| H1 | No fallback plan if `sales-0-01` spike fails | Todd Hintzmann, 2026-04-27 | Spike is 3-day timeboxed. If it fails, the feature is paused and an eTailPet support inquiry is raised before any Sprint 1 work begins. The sprint plan's gating on `sales-0-01` acceptance is the implicit fallback mechanism. |
| H2 | Fire-and-forget: trigger success ≠ email delivered; silent delivery failures undetected in v1 | Todd Hintzmann, 2026-04-27 | emailfetcher inbox monitoring and delivery alerting is deferred to a follow-on observability story. Operator is aware that `trigger_exhausted` is the only automated alert; email non-delivery requires manual inbox spot-check in v1. |

---

## Party-Mode Challenge

**Perturabo (Architect):** The fire-and-forget model is architecturally naive for a data pipeline. Trigger success means you got a 200. It does not mean eTailPet's email server sent anything, it does not mean the email reached the inbox, and it does not mean emailfetcher processed it. You have a critical observability gap in the first 10 minutes after every trigger. If email delivery breaks silently for a week, you have a week's gap in sales data with no alert and no automatic retry. The PRD needs to acknowledge this explicitly and either own a monitoring story or explicitly defer it with a documented acceptance of the risk.

**Inquisitor Greyfax (Analyst):** My concern is the data pipeline ownership vacuum. The PRD justifies this feature with Kaylee SKU-velocity tools and ordering workflow improvements. But the pipeline terminates at emailfetcher receiving an xlsx. There is no feature that parses the xlsx, no feature that writes it to a database table, no feature that wires it to Kaylee. The business case says "unlocks SKU-level velocity for Kaylee" — but nothing planned actually closes that loop. This is a data pipeline with no exit. What is the timeline for the next stage?

**Watch-Captain Artemis (QA):** The spike story `sales-0-01` is the entire validation gate for this feature. But the spike acceptance criteria don't confirm all the things the downstream stories assume. The stories assume: (1) LegacyClient OAuth2 form is correct, (2) the trigger GET returns 200, (3) a sales-line xlsx email arrives, (4) that email is identifiable by subject, (5) the xlsx has known column headers. If the spike only confirms (1) and (2) but eTailPet delays sending the email 24 hours, or the attachment is a CSV not xlsx, the Sprint 1 stories start on false premises. The spike needs explicit pass/fail criteria for all five of those confirmation points before any Sprint 1 story is started.

---

## Gaps You May Not Have Considered

1. **What if the spike confirms auth but the email never arrives during the spike timebox?** Do you wait 24 hours for confirmation, or does the spike fail? What is the maximum wait time before declaring `sales-0-01` failed?
2. **Does the www API require a TO_EMAIL parameter, and if so, where does that value come from?** Is it a Vault secret, a Helm value, or a fixed retailer setting in eTailPet's configuration?
3. **If TRIGGER_LOOKBACK_DAYS=7 runs daily, is eTailPet deduplicating server-side?** Will emailfetcher receive 7 days of overlapping sales data every day, and if so, does the downstream pipeline handle duplicate reports?
4. **Who owns the xlsx → DB pipeline stage?** Is there a story, feature, or person responsible for consuming the emailfetcher output? Or does the xlsx sit processed-but-discarded until a follow-on feature is planned?
5. **Can this feature be tested in staging without affecting the production eTailPet inbox?** If staging and production share the same eTailPet retailer credentials and inbox, a staging trigger sends real emails to a real inbox with no isolation.

---

## Open Questions Surfaced

These should be reflected in `prd.md` `open_questions` or resolved before TechPlan completion:

1. If `sales-0-01` fails (auth rejected or email not delivered), what is the decision: abandon, re-scope, or escalate to eTailPet support?
2. Does the www trigger require a `to_email` or equivalent parameter, or is the recipient a fixed retailer setting?
3. What is the email subject line pattern from eTailPet for `report_type=sales-line`? Is it deterministic enough for an exact emailfetcher routing rule?
4. Does eTailPet deduplicate report emails server-side when the same date range is requested on consecutive days?
5. Is there a staging-isolated eTailPet schema or separate test inbox available?
6. What is the planned next feature that parses the xlsx, stores data, and enables Kaylee SKU-velocity tools?
