---
name: uc-privileged-access-decision
org: universal-cake
lang: en
type: decision
version: 0.3.0
date: 2026-11-16
---

# Privileged access for raw and onboarded nodes

Version: 0.3.0

Status: Proposed

Author: Christopher Steel

This decision records why privileged access works the way [the privileged access policy](../node-policies/privileged-access/uc-privileged-access-policy.md) states, which options were rejected, and what it changes in the node-onboarding-pipeline contracts. The policy holds the requirements themselves.

## Context

The raw node contract (0.2.0) grants `initial_username` root through sudo with its password (RN-ACC-04), allows password login over SSH as a recovery path (RN-ACC-03), and says ingress never needs password hashes or other secrets. The onboarding egress contract (0.2.1) then makes sudo passwordless for `initial` (OE-CON-02).

That left two problems. Onboarding could not obtain root on a contract node, because the controller held the SSH key but not the password. And passwordless sudo is a finding under every hardening baseline the policy follows <a id="cite-disa-2026"></a>[(Defense Information Systems Agency [DISA], 2026)](#ref-disa-2026).

The goal set for this work is that normalized nodes conform to NIST SP 800-53 and to every authority that implements or echoes it, for privileged access and authentication.

## Options considered

### Option A: the controller holds the password, and sudo keeps asking for it

Chosen. It meets re-authentication (IA-11) and, with key-only SSH, multi-factor authentication for remote privileged access (IA-2(1)) <a id="cite-jtf-2020"></a>[(Joint Task Force, 2020)](#ref-jtf-2020). Its cost is a secret store in the controller, which the controller needs anyway.

### Option B: producers grant passwordless sudo at handoff

Rejected. Simpler, and close to cloud-init conventions, but it fails IA-11 on every node, and its only accepted form under the STIGs, a documented administrative group with multi-factor authentication, adds more than it saves.

### Console access

A password alone at the console would be single-factor access to a privileged account. That fails IA-2(1) and its Canadian counterpart, and it would bring in the PCI DSS rule that single-factor passwords change every 90 days. Adding a FIDO2 key at the console removes the single-factor case <a id="cite-pcissc-nd"></a>[(PCI Security Standards Council, n.d.)](#ref-pcissc-nd). Until a second key is held, this is covered by exception EX-0001.

### Password expiry

The RHEL 9 STIG requires 60-day expiry, and NIST SP 800-63B-4 advises against scheduled changes <a id="cite-nist-2025"></a>[(National Institute of Standards and Technology, 2025)](#ref-nist-2025). Automated replacement by the controller satisfies the first to the letter and the second in intent, and is recorded in the policy as its one deviation.

## Decision

The policy, version 0.1.0, applies to normalized nodes. In short:

- `initial` is a bootstrap account for the pipeline only, and normalization deletes it.
- Each privileged account belongs to one person or one automation identity, and `root` is locked.
- sudo always asks for the caller's own password, with no caching in `stage` or `prod`.
- SSH is key-only, and console login takes a password and a FIDO2 key.
- Passwords are generated, unique per account and node, and replaced automatically.
- Keys and host keys are registered and have a lifecycle.
- Strict values are the defaults in code, and only `dev` may relax them, through parameters.

Before normalization, the pipeline contracts below apply the parts of the policy that a raw or onboarded node can meet.

## Contract changes

### Raw node contract, 0.2.0 to 0.3.0

Incompatible, because RN-ACC-03 is retired.

| Change | Clause or field |
| --- | --- |
| Add the shared parameter `initial_password`, the secret ingress holds in its secret store. The producer receives only its hash. | Shared parameters |
| Replace "Ingress never needs them" with: ingress holds `initial_password`; every other secret belongs to the producer | Shared parameters |
| Retire: `initial_username` can log in over SSH with its password | RN-ACC-03 |
| Add: `initial_username` can log in at the console with its password, as the recovery path during the raw window | RN-ACC-06 |
| Add: no sudoers rule contains `NOPASSWD` or `!authenticate` | RN-ACC-07 |
| Add: `initial_password` is at least 15 characters | RN-ACC-08 |
| Add: the SSH server accepts public key authentication only | RN-SSH-03 |
| Add the field `ssh_host_keys`, a list of `type` and SHA256 `fingerprint` | Confirmation |
| Add the field `ssh_password_auth`, measured from the running SSH server | Confirmation |
| Add the field `sudo_nopasswd`, whether any sudoers rule grants it | Confirmation |

A producer builds nodes before their hostnames exist, so `initial_password` is shared per build batch. Ingress admits a node only when `ssh_password_auth` and `sudo_nopasswd` are false and the host key fingerprints match the connection.

### Onboarding egress contract, 0.2.1 to 0.3.0

Incompatible, because OE-CON-02 is retired.

| Change | Clause |
| --- | --- |
| Retire: `initial` obtains root through sudo without a password | OE-CON-02 |
| Add: `initial` obtains root through sudo with its password, held in the controller's secret store. Tested by running `id` with become. | OE-CON-05 |
| Add: `initial` has a password unique to the node, replacing the batch password. Converged. | OE-ACC-03 |
| Add: no sudoers rule contains `NOPASSWD` or `!authenticate`. Tested. | OE-ACC-04 |
| Add: the SSH server accepts public key authentication only. Tested with `sshd -T`. | OE-SSH-03 |
| Add "Become password: from the secret store, per node" to what normalization receives | What normalization receives |

### Normalization

Normalization has no contract yet. Its egress is compliance with the policy: it creates the automation and administrator accounts, deletes `initial`, locks `root`, and applies the console, password and key clauses. A normalization egress contract can later state this as clauses that point to the policy's.

## Changelog

| Version | Status | Notes |
| --- | --- | --- |
| 0.3.0 | Proposed | Requirements moved to uc-privileged-access-policy 0.1.0. Scope set to normalized nodes, with the pipeline contracts applying what they can earlier. `initial` recorded as a bootstrap account deleted at normalization. Console FIDO2 moved out of the raw node contract into the policy, under exception EX-0001 until a second key is held. Password expiry deviation recorded. |
| 0.2.0 | Proposed | Overview, goal and compliance scope added. Console login takes a FIDO2 key. PCI DSS 8.3.6 and 8.3.9 added. |
| 0.1.0 | Proposed | First version |

## References

<a id="ref-disa-2026"></a>
Defense Information Systems Agency. (2026). *Red Hat Enterprise Linux 9 security technical implementation guide* (Version 2, Release 8). https://dl.dod.cyber.mil/wp-content/uploads/stigs/zip/U_RHEL_9_V2R8_STIG.zip
[Return to citation](#cite-disa-2026)

<a id="ref-jtf-2020"></a>
Joint Task Force. (2020). *Security and privacy controls for information systems and organizations* (NIST Special Publication 800-53, Rev. 5). National Institute of Standards and Technology. https://doi.org/10.6028/NIST.SP.800-53r5
[Return to citation](#cite-jtf-2020)

<a id="ref-nist-2025"></a>
National Institute of Standards and Technology. (2025). *Digital identity guidelines: Authentication and authenticator management* (NIST Special Publication 800-63B-4). https://pages.nist.gov/800-63-4/sp800-63b/authenticators
[Return to citation](#cite-nist-2025)

<a id="ref-pcissc-nd"></a>
PCI Security Standards Council. (n.d.). *Do PCI DSS requirements 8.3.9 and 8.3.10.1 apply to all system components?* (FAQ 1590). Retrieved November 16, 2026, from https://www.pcisecuritystandards.org/faqs/1590
[Return to citation](#cite-pcissc-nd)
