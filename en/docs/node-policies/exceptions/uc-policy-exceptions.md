---
name: uc-policy-exceptions
org: universal-cake
lang: en
type: register
version: 0.1.0
date: 2026-11-16
---

# Policy exceptions register

Version: 0.1.0

Author: Christopher Steel

This register records every case where nodes in `stage` or `prod` depart from a node policy clause. It serves every policy under `node-policies/`. An exception is temporary: it covers something not yet in place, or a development activity that must run on a staging or production node for a limited time.

Two things are not exceptions. Development values that a policy's parameters table allows are part of the policy. A place where a policy deliberately departs from an outside authority is recorded in that policy's "Deviations from authorities" section.

## Fields

| Field | Meaning |
| --- | --- |
| ID | `EX-` and a four-digit number. Never reused. |
| Clauses | The policy clauses departed from |
| Scope | The environments, groups or nodes covered |
| Reason | Why the clause cannot be met yet |
| Compensating controls | What reduces the risk meanwhile |
| Approver | Who accepted the risk |
| Opened | Date approved |
| Expires | Date after which the clause counts as failing again |
| Status | `open`, `closed` (the clause is now met) or `expired` |

## Using an exception

1. Add a row to the register, with an expiry no more than 90 days away.
2. List the ID in `uc_pa_exceptions` for the nodes in scope, in the inventory.
3. Audits then report the clause as excepted on those nodes, and as failing after the expiry date.
4. When the clause is met, set the status to `closed` and remove the ID from the inventory.

Renewing an exception means a new row with a new ID, so that the history shows how long a gap stayed open.

## Register

| ID | Clauses | Scope | Reason | Compensating controls | Approver | Opened | Expires | Status |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| EX-0001 | PA-CON-01, PA-CON-02 | `stage` and `prod`, all nodes | One FIDO2 key held. A second is needed before console MFA can be enforced without risking lockout. | Console password of 15 or more characters; `root` locked; physical access to nodes restricted; SSH remains key-only with sudo password | Christopher Steel | 2026-11-16 | 2027-02-14 | open |

## Changelog

| Version | Notes |
| --- | --- |
| 0.1.0 | First version, with EX-0001 |
