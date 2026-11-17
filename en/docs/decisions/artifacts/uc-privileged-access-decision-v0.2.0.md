---
name: uc-privileged-access-decision
org: universal-cake
lang: en
type: decision
version: 0.2.0
date: 2026-11-16
---

# Privileged access for raw and onboarded nodes

Version: 0.2.0

Status: Proposed

Author: Christopher Steel

This decision settles how anyone, person or controller, obtains root on a node. It applies for the node's whole life, from the moment a producer hands over a raw node until the node is retired. The node-onboarding-pipeline contracts are where it is first enforced, and the changes they need are listed below.

## Overview

### The decision

- sudo always asks for the invoking user's password. No sudoers rule contains `NOPASSWD` or `!authenticate`.
- SSH accepts public keys only. Password login over SSH is disabled, and `root` cannot log in over SSH.
- The password remains a recovery path at the console, where it is paired with a FIDO2 security key, so that every path to root takes two factors.
- The confirmation reports the node's SSH host key fingerprints, and ingress verifies them before it trusts the node.

### The authorities it follows

NIST SP 800-53 is the reference. The DISA STIGs implement it as configuration checks, and the CIS Benchmarks carry the same check for Debian, for which DISA publishes no STIG. CIS Controls v8, the Canadian Centre for Cyber Security's ITSP.10.033 and ITSM.10.094, PCI DSS v4.0.1 and ISO/IEC 27001:2022 reach the same requirements independently. ITSP.10.033 is Canada's counterpart of NIST SP 800-53, which matters for smaller government clients.

### Judgement calls

- A producer builds nodes before their hostnames exist, so the password is shared per build batch during the raw window only. Onboarding replaces it with a password unique to the node.
- A password alone at the console would be single-factor access to a privileged account. That falls short of NIST SP 800-53 Rev. 5 IA-2(1) and ITSP.10.033, and it would also oblige 90-day password changes under PCI DSS, which NIST SP 800-63B-4 forbids. Adding a FIDO2 key at the console removes the single-factor case, and with it both conflicts.
- Passwords are at least 15 characters, so that they hold up even where a password is the only factor, for example in a rescue boot.
- Both contract changes are incompatible, because each retires a clause: RN-ACC-03 (password login over SSH) and OE-CON-02 (passwordless sudo).

## Goal and compliance scope

The goal is that every node, at every stage, conforms to each authority below for privileged access and authentication. Each row in the authorities table is met by a decision here, with no residual gap.

Compliance with a whole framework is wider than this decision. NIST SP 800-53, ITSP.10.033 and PCI DSS are full control sets, and ISO/IEC 27001 certifies an organization's management system, not a node. This decision covers their access control and authentication requirements for Linux nodes, and makes those parts of an assessment straightforward to evidence. Other areas, such as logging, patching and the firewall, need decisions of their own.

## Context

The raw node contract (0.1.0) grants `initial_username` root through sudo with its password (RN-ACC-04), allows password login over SSH as a recovery path (RN-ACC-03), and says ingress never needs password hashes or other secrets. The onboarding egress contract (0.2.0) then makes sudo passwordless for `initial` (OE-CON-02).

That leaves two problems. Onboarding cannot obtain root on a contract node, because the controller holds the SSH key but not the password. And the onboarded state, passwordless sudo, is a finding under every hardening baseline listed below.

Two options were compared: give the controller the `initial` password (sudo keeps its password), or have producers grant passwordless sudo at handoff. The second is simpler but fails the re-authentication and multi-factor requirements below, on raw nodes and onboarded nodes alike.

## Decision

### Privilege escalation always takes a password

On every node, at every stage, a user obtains root through sudo with their own password. No sudoers rule contains `NOPASSWD` or `!authenticate`, and sudo never asks for the root password. Ansible supplies the password as `ansible_become_password` from the controller's vault.

### Network login is by key only

The SSH server accepts public key authentication only. Password and keyboard-interactive authentication are disabled. `root` keeps a password and cannot log in over SSH.

Remote root therefore needs two factors: the private key (something you have) and the sudo password (something you know). Neither alone is enough.

### Console login takes a password and a security key

At the console, physical or out-of-band (a hypervisor console, a BMC, a serial line), login takes the user's password and a FIDO2 security key, through `pam_u2f` in the console's PAM stack only. sudo is not changed, so Ansible over SSH is unaffected.

The producer receives the operator's key registrations, made with `pamu2fcfg`, as an input. They are public data, not secrets. At least two keys are registered, so that losing one does not lock out the console.

A rescue boot from external media bypasses the installed system's authentication. It is governed by physical security and, where the hardware allows, firmware and boot passwords, not by this decision.

### Password strength

Passwords are generated by the controller, never chosen by hand, and are at least 15 characters. Inside multi-factor authentication 8 would suffice, but 15 is the floor wherever a password could end up as the only factor. Generated passwords contain both letters and digits, which meets PCI DSS without imposing composition rules on anyone, as NIST SP 800-63B-4 requires. Passwords are changed when compromise is suspected, not on a schedule.

### Password uniqueness

A producer such as uc-ventoy builds nodes before their hostnames exist, so one password per build batch is the most it can do. Onboarding replaces it with a password unique to the node, stored in the vault under that node. A batch password is therefore only ever shared during the raw window.

### Host keys are verified, not trusted on first use

The confirmation reports the node's SSH host key fingerprints. Ingress compares them with the fingerprints it reached the node with before it trusts the connection, and writes them to the controller's `known_hosts`.

### The ingress key has a lifecycle

The ingress key pair is an automation identity. Where its private key is kept, how it is rotated, and how its public key is removed from a node are documented in uc-controller. Each node's `authorized_keys` holds only the keys the contracts name.

### Exceptions are explicit

Passwordless sudo is not used. If a future need arises, it will be a separate decision, limited to a documented administrative group that authenticates with multiple factors, which is the only exception the current STIGs allow.

## Factors by access path

| Access path | Factors | Result |
| --- | --- | --- |
| SSH as `initial`, then sudo | Private key, then password | Multi-factor |
| SSH with a password | Disabled | Not possible |
| SSH as `root` | Disabled | Not possible |
| Console, then sudo | Password and FIDO2 key, then password | Multi-factor |
| Stolen SSH key alone | Key | Unprivileged shell only |
| Stolen password alone | Password | Nothing |
| Stolen FIDO2 key alone | Key | Nothing |

## Authorities

| Authority | Requirement | How this decision meets it |
| --- | --- | --- |
| NIST SP 800-53 Rev. 5, IA-2(1) <a id="cite-jtf-2020"></a>[(Joint Task Force, 2020)](#ref-jtf-2020) | Multi-factor authentication for access to privileged accounts | Two factors on every path to root, network and console |
| NIST SP 800-53 Rev. 5, IA-11 [(Joint Task Force, 2020)](#ref-jtf-2020) | Re-authentication when privilege is escalated | sudo always asks for the password |
| NIST SP 800-53 Rev. 5, IA-5 [(Joint Task Force, 2020)](#ref-jtf-2020) | Authenticators managed and not shared | Generated passwords, unique per node after onboarding, held in the vault |
| NISTIR 7966 <a id="cite-ylonen-2015"></a>[(Ylonen et al., 2015)](#ref-ylonen-2015) | Public key authentication for automated access, a managed key lifecycle, and host key management against man-in-the-middle attacks | Key-only SSH for the controller, a documented key lifecycle, host key fingerprints in the confirmation |
| NIST SP 800-63B-4 <a id="cite-nist-2025"></a>[(National Institute of Standards and Technology [NIST], 2025)](#ref-nist-2025) | At least 15 characters for a single-factor password, at least 8 inside multi-factor authentication, no composition rules, no scheduled changes | 15 or more, generated, changed only on suspected compromise |
| DISA STIGs for Ubuntu and RHEL <a id="cite-disa-2026a"></a>[(Defense Information Systems Agency [DISA], 2026a)](#ref-disa-2026a) | No `NOPASSWD` in sudoers, mapped to IA-11 | No `NOPASSWD` on any node |
| DISA STIG for RHEL 10 <a id="cite-disa-2026b"></a>[(DISA, 2026b)](#ref-disa-2026b) | `NOPASSWD` only for a documented administrative group that uses multi-factor authentication | The only exception path, and not taken |
| CIS Debian Linux Benchmark <a id="cite-cis-nd"></a>[(Center for Internet Security [CIS], n.d.)](#ref-cis-nd) | Users must provide a password for privilege escalation | Met. DISA publishes no Debian STIG, so this benchmark is the Debian reference. |
| CIS Critical Security Controls v8, safeguard 6.5 <a id="cite-cis-2021"></a>[(CIS, 2021)](#ref-cis-2021) | Multi-factor authentication for all administrative access, where supported | Two factors on every path to root |
| ITSP.10.033, IA-2(1) <a id="cite-cccs-nd"></a>[(Canadian Centre for Cyber Security [CCCS], n.d.)](#ref-cccs-nd) | Strong multi-factor authentication for privileged accounts, whatever the type of access | Two factors at the console as well as over the network |
| ITSM.10.094, Top 10 IT security actions no. 3 <a id="cite-cccs-2022"></a>[(CCCS, 2022)](#ref-cccs-2022) | Manage and control administrative privileges, with multi-factor authentication | Met as above |
| PCI DSS v4.0.1, requirement 8.4.1 <a id="cite-pcissc-2024"></a>[(PCI Security Standards Council [PCI SSC], 2024)](#ref-pcissc-2024) | Multi-factor authentication for all non-console administrative access into the cardholder data environment | Met for every SSH path, should a client's node fall in scope |
| PCI DSS v4.0.1, requirements 8.3.6 and 8.3.9 [(PCI SSC, 2024)](#ref-pcissc-2024), with FAQ 1590 <a id="cite-pcissc-nd"></a>[(PCI SSC, n.d.)](#ref-pcissc-nd) | At least 12 characters with letters and digits. Passwords used as the only factor change every 90 days or are analysed dynamically, which does not apply where multi-factor authentication is used. | 15 or more with letters and digits. No path uses a password alone, so 8.3.9 does not apply. |
| ISO/IEC 27001:2022, Annex A 8.2 and 8.5 <a id="cite-iso-2022"></a>[(International Organization for Standardization [ISO], 2022)](#ref-iso-2022) | Privileged access rights restricted and managed, and secure authentication | One escalation path, re-authenticated, with recorded credentials |

ITSP.10.033 is the Cyber Centre's catalogue of security controls, its counterpart of NIST SP 800-53, and is the reference for Canadian public sector clients.

## Contract changes

### Raw node contract, 0.1.0 to 0.2.0

Incompatible, because RN-ACC-03 is retired.

| Change | Clause or field |
| --- | --- |
| Add the shared parameter `initial_password`, the secret ingress holds in its vault. The producer receives only its hash. | Shared parameters |
| Add the shared parameter `console_u2f_mappings`, the operator's FIDO2 key registrations. Public. | Shared parameters |
| Replace "Ingress never needs them" with: ingress holds `initial_password`; every other secret belongs to the producer | Shared parameters |
| Retire: `initial_username` can log in over SSH with its password | RN-ACC-03 |
| Add: `initial_username` can log in at the console with its password and a FIDO2 key registered in `console_u2f_mappings` | RN-ACC-06 |
| Add: no sudoers rule contains `NOPASSWD` or `!authenticate` | RN-ACC-07 |
| Add: `initial_password` is at least 15 characters | RN-ACC-08 |
| Add: the SSH server accepts public key authentication only | RN-SSH-03 |
| Add the field `ssh_host_keys`, a list of `type` and SHA256 `fingerprint` | Confirmation |
| Add the field `ssh_password_auth`, measured from the running SSH server | Confirmation |
| Add the field `sudo_nopasswd`, whether any sudoers rule grants it | Confirmation |
| Add the field `console_mfa`, whether the console PAM stack requires `pam_u2f` | Confirmation |

Ingress checks the new fields when it admits a node: `ssh_password_auth` and `sudo_nopasswd` false, `console_mfa` true, and the host key fingerprints matching the connection.

### Onboarding egress contract, 0.2.0 to 0.3.0

Incompatible, because OE-CON-02 is retired.

| Change | Clause |
| --- | --- |
| Retire: `initial` obtains root through sudo without a password | OE-CON-02 |
| Add: `initial` obtains root through sudo with its password, which the controller holds in its vault. Tested by running `id` with become. | OE-CON-05 |
| Add: `initial` has a password unique to the node, replacing the batch password. Converged. | OE-ACC-03 |
| Add: no sudoers rule contains `NOPASSWD` or `!authenticate`. Tested. | OE-ACC-04 |
| Add: console login requires the password and a registered FIDO2 key. Tested by reading the PAM stack. | OE-ACC-05 |
| Add: the SSH server accepts public key authentication only. Tested with `sshd -T`. | OE-SSH-03 |
| Add "Become password: from the vault, per node" to what normalization receives | What normalization receives |

The sudo row of the overlap table then reads as consistent.

## Implementation notes

### uc-controller

The vault holds `vault_initial_password` per node in `host_vars/<host>/vault.yml`, and per batch for raw nodes not yet onboarded. `group_vars` for the raw and onboarded groups set `ansible_become_password: "{{ vault_initial_password }}"`. The onboard playbook generates the node's own password, writes it to the vault, applies it, and only then drops the batch password. If any step fails, the batch password still works.

### uc-ventoy

`raw-node-input.yml` keeps `initial_password_hash`, now derived from the batch password in the controller's vault rather than kept on its own, and gains `console_u2f_mappings`. The installer disables SSH password authentication, writes no `NOPASSWD` rule, installs `libpam-u2f` and adds it to the console's PAM stack only, and adds the four new confirmation fields.

## Open points

Whether onboarding should also lock the console password once out-of-band recovery exists through another path, such as a hypervisor console or a BMC, is left for a later decision.

Which FIDO2 keys the operator uses, and how registrations are added and removed over a node's life, belongs in uc-controller's key lifecycle documentation.

## Changelog

| Version | Status | Notes |
| --- | --- | --- |
| 0.2.0 | Proposed | Overview, goal and compliance scope added. Scope widened to the node's whole life. Console login now takes a FIDO2 key as well as the password, which closes the single-factor gap and keeps PCI DSS 8.3.9 out of scope. PCI DSS 8.3.6 and 8.3.9 added to the authorities. |
| 0.1.0 | Proposed | First version: passwords for every escalation, key-only SSH, console recovery, host key verification, and the resulting changes to both contracts |

## References

<a id="ref-cccs-2022"></a>
Canadian Centre for Cyber Security. (2022). *Top 10 IT security actions: No. 3 managing and controlling administrative privileges* (ITSM.10.094). Communications Security Establishment. https://cyber.gc.ca/en/guidance/top-10-it-security-actions-no3-managing-controlling-administrative-privileges-itsm10094
[Return to citation](#cite-cccs-2022)

<a id="ref-cccs-nd"></a>
Canadian Centre for Cyber Security. (n.d.). *Identification and authentication* (ITSP.10.033, Security control catalogue). Retrieved November 16, 2026, from https://cyber.gc.ca/en/guidance/cyber-security-privacy-risk-management/itsp10033/identification-authentication
[Return to citation](#cite-cccs-nd)

<a id="ref-cis-2021"></a>
Center for Internet Security. (2021). *CIS Critical Security Controls* (Version 8). https://www.cisecurity.org/controls/v8
[Return to citation](#cite-cis-2021)

<a id="ref-cis-nd"></a>
Center for Internet Security. (n.d.). *CIS Debian Linux 12 Benchmark v1.0.1, recommendation 5.2.4: Ensure users must provide password for privilege escalation* [Audit item]. Tenable. Retrieved November 16, 2026, from https://tenable.com/audits/items/CIS_Debian_Linux_12_v1.0.1_L2_Server.audit:f54d5fa82093a914e611bb4849720f12
[Return to citation](#cite-cis-nd)

<a id="ref-disa-2026a"></a>
Defense Information Systems Agency. (2026a). *Canonical Ubuntu 22.04 LTS security technical implementation guide* (Rule V-274860). STIG Viewer. https://www.stigviewer.com/stigs/canonical_ubuntu_2204_lts/2026-02-06/finding/V-274860
[Return to citation](#cite-disa-2026a)

<a id="ref-disa-2026b"></a>
Defense Information Systems Agency. (2026b). *Red Hat Enterprise Linux 10 security technical implementation guide* (Rule V-281211). STIG Viewer. https://trial.stigviewer.com/stigs/red_hat_enterprise_linux_10/2026-03-11/finding/V-281211
[Return to citation](#cite-disa-2026b)

<a id="ref-iso-2022"></a>
International Organization for Standardization. (2022). *Information security, cybersecurity and privacy protection: Information security management systems: Requirements* (ISO/IEC Standard No. 27001:2022). https://www.iso.org/standard/27001
[Return to citation](#cite-iso-2022)

<a id="ref-jtf-2020"></a>
Joint Task Force. (2020). *Security and privacy controls for information systems and organizations* (NIST Special Publication 800-53, Rev. 5). National Institute of Standards and Technology. https://doi.org/10.6028/NIST.SP.800-53r5
[Return to citation](#cite-jtf-2020)

<a id="ref-nist-2025"></a>
National Institute of Standards and Technology. (2025). *Digital identity guidelines: Authentication and authenticator management* (NIST Special Publication 800-63B-4). https://pages.nist.gov/800-63-4/sp800-63b/authenticators
[Return to citation](#cite-nist-2025)

<a id="ref-pcissc-2024"></a>
PCI Security Standards Council. (2024). *Payment Card Industry Data Security Standard: Requirements and testing procedures* (Version 4.0.1). https://www.pcisecuritystandards.org/document_library/
[Return to citation](#cite-pcissc-2024)

<a id="ref-pcissc-nd"></a>
PCI Security Standards Council. (n.d.). *Do PCI DSS requirements 8.3.9 and 8.3.10.1 apply to all system components?* (FAQ 1590). Retrieved November 16, 2026, from https://www.pcisecuritystandards.org/faqs/1590
[Return to citation](#cite-pcissc-nd)

<a id="ref-ylonen-2015"></a>
Ylonen, T., Turner, P., Scarfone, K., & Souppaya, M. (2015). *Security of interactive and automated access management using Secure Shell (SSH)* (NISTIR 7966). National Institute of Standards and Technology. https://doi.org/10.6028/NIST.IR.7966
[Return to citation](#cite-ylonen-2015)
