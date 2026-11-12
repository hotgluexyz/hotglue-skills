---
name: validating-integration-specs
description: Validates integration business rules against service API and UI documentation, representative data, and deployed implementation. Use when creating or reviewing integration specs, mappings, or claimed gaps before proposing fixes.
---

# Validating Integration Specs

Validate the business rule first, its applicability to real data second, and the implementation third.
Treat specs, existing code, user diagnoses, and automated reviews as claims to investigate.
Review one entity and one direction at a time; honor exclusions and documented accepted limitations.
Review is read-only unless changes or test runs are explicitly authorized.

## 1. Establish the execution model

- For Hotglue, load `working-with-hotglue` and `understanding-hotglue-jobs` (Docs MCP + incremental/snapshot traps).
- Use `running-hotglue-remote-jobs` and `managing-available-connectors` when relevant.
- Identify tenant, environment, direction, tap, ETL, target, and deployed versions.
- Inspect connector definitions, selected fields/streams, tenant mappings, filters, and job settings.
- Validate the business outcome before treating incremental extraction as a spec violation; determine whether the intended outcome needs deltas, cached context, or genuine full refreshes. If the spec explicitly requires full refreshes, explain the tradeoff and confirm intent before changing it.
- Do not assume deltas capture deletions or related-stream changes; verify the actual API/bookmark contract.

## 2. Validate the spec against API and UI behavior

- Read the relevant source and destination API docs, UI docs, and configured connector contract.
- Verify field meaning, identity, relationships, lifecycle states, dates, limits, and upsert behavior.
- Distinguish codes from labels, tax categories from registration IDs, and status from approval. Check tenant master-data/configuration where fields are configurable rather than fixed enums.
- API acceptance proves a payload is accepted, not that it represents a documented UI workflow. Do not treat schema nullability, a hypothetical example, or another product's behavior as proof.
- If API and UI differ, record the distinction and validate the tenant's actual business workflow. Mark an invalid or ambiguous requirement as a spec issue before judging implementation compliance; state missing documentation or unverified semantics explicitly — do not fill gaps with guesses.

## 3. Validate applicability using representative data

- Use supplied real data and authorized read-only API probes; keep secrets and customer details private. Follow the user's retention instructions; otherwise agree on retention and delete temporary data afterward.
- Check types, uniqueness, nulls, actual statuses, date boundaries, and relationship cardinality.
- Report total, matched, unmatched, and duplicate counts for proposed joins; test both join directions as needed.
- Prefer documented identifiers with observed relationships over names or speculative matching rules.
- Separate a bad mapping from isolated source-data exceptions; quantify exceptions before rejecting a rule. A single unmatched record does not invalidate the best documented relationship. If exceptions are rare and no better key exists, propose reviewed data correction, not a new mapping heuristic. Never silently discard exceptions, change identity, or mutate customer records without authorization.
- An empty sandbox or artificial inputs cannot prove tenant-relevant gaps or all scenarios; populate sandbox data only with authorization, using documented UI-valid states. Do not generalize one tenant's data into an API-wide guarantee.

## 4. Validate the implementation against the validated rule

- Trace source response through tap output, filters, ETL joins, export, target payload, and saved state.
- Verify installed branches/configuration rather than assuming local code is deployed.
- Inspect target preprocessing and request methods; an emitted field may be dropped or transformed.
- Assess first import, changed-record sync, no-change sync, and repeat-write identity where relevant. Test risky behavior with asymmetric inputs, actual states, date boundaries, and observed data formats.
- Use local replay or existing smoke fixtures before proposing remote runs or target writes. Follow the repository's test pattern; do not introduce unit tests, frameworks, or CI changes by default.
- Keep speculative compatibility guards, unrelated cleanup, and unrequested lifecycle handling out of fixes. Put generic extraction/loading in tap/target; cross-system business mapping belongs in custom ETL.
- A tap+ETL success does not validate target writes; distinguish each validated stage.

## 5. Report only actionable findings

- Unless asked otherwise, return gaps only, quoting the original spec row followed by a short explanation.
- Classify each as confirmed implementation gap, spec issue, source-data exception, or unverified hypothesis. For each gap, identify the owner: tap, target, custom ETL, connector definition, or tenant configuration/data.
- Give the decisive source permalink or probe result and the smallest evidence-backed next step. Keep hypothetical risks separate from reproduced failures; never claim duplicates or data loss without evidence. Do not repeat accepted limitations or documented backlog items as newly discovered gaps.
- State what was not validated and whether any code, shared settings, jobs, or target writes changed.
