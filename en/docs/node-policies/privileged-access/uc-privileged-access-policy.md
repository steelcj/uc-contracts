---
name: uc-privileged-access-policy
org: universal-cake
lang: en
type: policy
version: 0.1.0
date: 2026-11-16
---

# Privileged access policy

Version: 0.1.0

Status: Draft

Author: Christopher Steel

This policy states what must always be true of privileged access on a normalized node: who may obtain root, how they authenticate, and how the credentials involved are kept. Every clause names the authorities it satisfies and how it is checked on a node. The reasons behind it are recorded in [uc-privileged-access-decision](../../decisions/uc-privileged-access-decision.md).

## Overview

- Every privileged account belongs to one person or one automation identity. The bootstrap account `initial` is gone once a node is normalized, and `root` is locked.
- sudo always authenticates the person or identity running it, with their own password, and never caches it in production.
- SSH accepts public keys only, and never for `root`.
- Console login for administrators takes a password and a FIDO2 security key.
- Passwords are generated, long, held in a secret store or password manager, and rotated automatically.
- SSH keys, FIDO2 registrations and host keys are recorded in a register and have a lifecycle.
- A node departs from a clause only under a recorded, expiring exception.

## Scope

### Nodes

The policy applies to every normalized node, whatever its kind (metal, virtual machine or container) and whatever produced it. Normalization is the stage that must leave a node compliant, and audits check normalized nodes against this policy.

Before normalization, the node-onboarding-pipeline contracts govern the node. They move it towards this policy, and several of their clauses enforce parts of it early, but a node is not claimed compliant until it is normalized.

### Distributions

Debian 13, Ubuntu 24.04 LTS and Rocky Linux 9. Rocky Linux 9 is checked against the RHEL 9 STIG, and Debian against the CIS Debian Linux Benchmark, since DISA publishes no STIG for either.

### Provider hosts

Privileged access to a provider host, such as an Incus host, a hypervisor or a BMC, is privileged access to every node it runs: `incus exec` gives root in a guest without any authentication inside it. A provider host is therefore a node under this policy, and access to it is governed by the same clauses. Preparing a provider host before its guests enter ingress is pre-ingress work, outside this policy, but the host itself is not.

## Environments

Every node belongs to one environment, set in the inventory as `uc_environment`.

| Environment | Must comply | Compliance claimed | Notes |
| --- | --- | --- | --- |
| `prod` | Yes | Yes | Every parameter at its production value |
| `stage` | Yes | Yes | Same values as production, so that what is tested is what runs |
| `dev` | No | Never | Parameters may take their development values. Dev nodes hold no client data. |

The code implements every clause in every environment. Only the parameters below change between environments, and their defaults in code are the production values. A development value is set in the inventory for the `dev` environment, never in the code.

## Parameters

| Parameter | `prod` and `stage` | `dev` | Clause |
| --- | --- | --- | --- |
| `uc_pa_sudo_timestamp_timeout` | `0` (authenticate every time) | Up to `15` minutes | PA-ESC-02 |
| `uc_pa_sudo_logfile` | `/var/log/sudo.log` | Same | PA-ESC-03 |
| `uc_pa_console_mfa` | `true` | `true` or `false` | PA-CON-01 |
| `uc_pa_password_min_length` | `15` | Same | PA-PWD-01 |
| `uc_pa_password_max_age_days` | `60` | Up to `365` | PA-PWD-03 |
| `uc_pa_key_max_age_days` | `365` | Same | PA-KEY-03 |
| `uc_pa_admin_user` | `admin` | Same | PA-ACC-02 |
| `uc_pa_automation_user` | Set per controller | Same | PA-ACC-02 |

## Accounts

| Class | Account | Held by | Lifetime |
| --- | --- | --- | --- |
| Bootstrap | `initial` | The controller, during the pipeline | Created by the producer, deleted at normalization |
| Automation | `uc_pa_automation_user` | One controller | For as long as the controller manages the node |
| Administrator | `uc_pa_admin_user` | One named person | For as long as that person administers the node |
| Superuser | `root` | Nobody | Locked |

When a second person administers nodes, each person gets their own administrator account. A single `admin` account shared between people would be a group authenticator, which the authorities below do not allow.

## Authorities

Each clause cites these by their short label, with the rule identifier where one exists.

| Label | Authority |
| --- | --- |
| 800-53 | NIST SP 800-53 Rev. 5 <a id="cite-jtf-2020"></a>[(Joint Task Force, 2020)](#ref-jtf-2020) |
| 800-63B | NIST SP 800-63B-4 <a id="cite-nist-2025"></a>[(National Institute of Standards and Technology [NIST], 2025)](#ref-nist-2025) |
| 7966 | NISTIR 7966 <a id="cite-ylonen-2015"></a>[(Ylonen et al., 2015)](#ref-ylonen-2015) |
| STIG U24 | DISA STIG for Ubuntu 24.04 LTS <a id="cite-disa-2026a"></a>[(Defense Information Systems Agency [DISA], 2026a)](#ref-disa-2026a) |
| STIG R9 | DISA STIG for RHEL 9, applied to Rocky Linux 9 <a id="cite-disa-2026b"></a>[(DISA, 2026b)](#ref-disa-2026b) |
| STIG R10 | DISA STIG for RHEL 10, for its exception wording only <a id="cite-disa-2026c"></a>[(DISA, 2026c)](#ref-disa-2026c) |
| CIS Debian | CIS Debian Linux 12 Benchmark, applied to Debian 13 until a Debian 13 benchmark is adopted <a id="cite-cis-nda"></a>[(Center for Internet Security [CIS], n.d.-a)](#ref-cis-nda) |
| CIS Ubuntu | CIS Ubuntu Linux Benchmark <a id="cite-cis-ndb"></a>[(CIS, n.d.-b)](#ref-cis-ndb) |
| CIS Controls | CIS Critical Security Controls v8 <a id="cite-cis-2021"></a>[(CIS, 2021)](#ref-cis-2021) |
| ITSP.10.033 | Canadian Centre for Cyber Security control catalogue <a id="cite-cccs-nd"></a>[(Canadian Centre for Cyber Security [CCCS], n.d.)](#ref-cccs-nd) |
| PCI DSS | PCI DSS v4.0.1 <a id="cite-pcissc-2024"></a>[(PCI Security Standards Council [PCI SSC], 2024)](#ref-pcissc-2024), with FAQ 1590 <a id="cite-pcissc-nd"></a>[(PCI SSC, n.d.)](#ref-pcissc-nd) |

## Clauses

### Accounts (PA-ACC)

| ID | Requirement | Authorities | Check |
| --- | --- | --- | --- |
| PA-ACC-01 | The bootstrap account `initial` does not exist | 800-53 AC-2 | `getent passwd initial` returns nothing |
| PA-ACC-02 | Every member of the administrative group (`sudo` on Debian and Ubuntu, `wheel` on Rocky) is an account in the account register, held by exactly one person or one automation identity | 800-53 AC-2, IA-2; STIG U24 UBTU-24-400110 (no group authenticators) | Group members compared with the register |
| PA-ACC-03 | No account other than those in PA-ACC-02 can obtain root, through sudo or otherwise | 800-53 AC-6 | sudoers parsed for every grant; `su` restricted to the administrative group |
| PA-ACC-04 | `root` is locked. Its password is generated and kept in the password manager, for unlocking during recovery only. | STIG U24 UBTU-24-400110; 800-53 IA-2 | `passwd -S root` shows `L` |
| PA-ACC-05 | Access to a provider host's management interface (`incus`, a hypervisor console, a BMC) is limited to the accounts in PA-ACC-02 on that host | 800-53 AC-6 | Membership of the provider's administrative group (`incus-admin` for Incus) compared with the register |

### Escalation (PA-ESC)

| ID | Requirement | Authorities | Check |
| --- | --- | --- | --- |
| PA-ESC-01 | sudo authenticates whoever runs it with their own password. No sudoers rule contains `NOPASSWD` or `!authenticate`, and `rootpw`, `targetpw` and `runaspw` are off. | 800-53 IA-11; STIG U24 V-274868; STIG R9 RHEL-09-611085; CIS Debian 5.2.4 | `grep -ri` over `/etc/sudoers` and `/etc/sudoers.d` finds none |
| PA-ESC-02 | sudo credentials are cached for at most `uc_pa_sudo_timestamp_timeout` minutes, defined once | 800-53 IA-11; STIG R9 RHEL-09-432015 (`0`); CIS Ubuntu 5.2.6 (15 or less) | Exactly one `timestamp_timeout` setting, with the parameter's value |
| PA-ESC-03 | Every use of sudo is logged to `uc_pa_sudo_logfile` as well as to the system log | 800-53 AU-12; CIS Debian 5.2.3 | `Defaults logfile` set; a test command appears in the log |

The `0` timeout costs Ansible nothing: it supplies the password for every task. It is the strictest value, and it also satisfies CIS, which allows up to 15 minutes.

### Network login (PA-NET)

| ID | Requirement | Authorities | Check |
| --- | --- | --- | --- |
| PA-NET-01 | The SSH server accepts public key authentication only: password, keyboard-interactive and empty-password logins are off | 800-53 IA-2(1); 7966; STIG U24 UBTU-24-300031; PCI DSS 8.4.1 | `sshd -T` shows `passwordauthentication no`, `kbdinteractiveauthentication no`, `permitemptypasswords no` |
| PA-NET-02 | `root` cannot log in over SSH | STIG R9 RHEL-09-255045; 800-53 IA-2 | `sshd -T` shows `permitrootlogin no` |
| PA-NET-03 | Every account's `authorized_keys` holds only keys that the key register assigns to that account on that node | 7966; 800-53 AC-2, IA-5 | Keys on the node compared with the register |

Over the network, root therefore takes two factors: the private key, then the sudo password.

### Console (PA-CON)

| ID | Requirement | Authorities | Check |
| --- | --- | --- | --- |
| PA-CON-01 | When `uc_pa_console_mfa` is true, console login for an administrator takes their password and a FIDO2 key registered to them, through `pam_u2f` in the console's PAM stack only | 800-53 IA-2(1); ITSP.10.033 IA-2(1); CIS Controls 6.5 | `pam_u2f` required in the `login` stack, and in a display manager's stack if one is installed; not in `sudo` or `sshd` |
| PA-CON-02 | Each administrator has at least two FIDO2 keys registered, so that losing one does not lock them out | 800-53 IA-5 | Key register lists two or more active keys per administrator |
| PA-CON-03 | Automation and bootstrap accounts cannot log in at the console | 800-53 AC-2 | `pam_access` denies them on `LOCAL` and on every tty |

A container has no console. Its console-equivalent is the provider host's management interface, covered by PA-ACC-05.

With the console clauses in force, no path to root uses a password alone, so the PCI DSS rule on single-factor passwords (8.3.9) does not apply, as its FAQ 1590 confirms.

### Passwords (PA-PWD)

| ID | Requirement | Authorities | Check |
| --- | --- | --- | --- |
| PA-PWD-01 | Passwords of privileged accounts are generated, never chosen by hand, are at least `uc_pa_password_min_length` characters, and contain letters and digits. No composition rules are imposed on people. | 800-63B; PCI DSS 8.3.6; 800-53 IA-5(1) | Generation recorded by the controller; `pwquality` sets the minimum length |
| PA-PWD-02 | Each privileged account's password is unique to that account on that node. Automation passwords live in the controller's secret store, administrator and `root` passwords in the password manager. | 800-53 IA-5 | Secret store and password manager hold one entry per account and node |
| PA-PWD-03 | Privileged passwords expire after `uc_pa_password_max_age_days`, and the controller replaces them before they expire, updating the secret store or password manager in the same run | STIG R9 RHEL-09-411015; 800-53 IA-5(1) | `chage -l` shows the maximum age; the last change is younger than it |
| PA-PWD-04 | A password is replaced at once on suspected compromise | 800-63B; 800-53 IA-5 | Incident record |

### Keys (PA-KEY)

| ID | Requirement | Authorities | Check |
| --- | --- | --- | --- |
| PA-KEY-01 | Every SSH key and FIDO2 registration that grants access is in the key register, with its owner, type, fingerprint, where the private part is kept, creation date, expiry and status | 7966; 800-53 AC-2, IA-5 | Register entries complete |
| PA-KEY-02 | SSH keys are Ed25519, or Ed25519 bound to a FIDO2 key (`ed25519-sk`) for people. An automation identity's private key never leaves its controller's secret store. | 7966; 800-53 IA-5(2) | Key types on nodes; register locations |
| PA-KEY-03 | Keys are replaced within `uc_pa_key_max_age_days`, at once on suspected compromise, and at once when their holder leaves | 7966; 800-53 IA-5 | Register expiry dates; no active key past expiry |
| PA-KEY-04 | A key retired in the register is removed from every node at the next run, and the run fails rather than leaves it in place | 7966; 800-53 AC-2 | No retired fingerprint on any node |

### Host keys (PA-HOST)

| ID | Requirement | Authorities | Check |
| --- | --- | --- | --- |
| PA-HOST-01 | The controller verifies a node's host keys against the fingerprints recorded at ingress, and refuses to connect when they differ | 7966; 800-53 IA-3 | `StrictHostKeyChecking yes` in the controller's SSH settings; `known_hosts` built from the register |
| PA-HOST-02 | A node's host keys change only on reinstall, which also changes its `machine_id`, and the change is recorded before the controller trusts the new keys | 7966 | Register history |

### Exceptions (PA-EXC)

| ID | Requirement | Authorities | Check |
| --- | --- | --- | --- |
| PA-EXC-01 | A node in `stage` or `prod` departs from a clause only under an exception in [the exceptions register](../exceptions/uc-policy-exceptions.md) | 800-53 CA-5, PM-4 | Every failing check maps to an open exception |
| PA-EXC-02 | Every exception has an expiry date. An expired exception counts as a failing clause. | 800-53 CA-5 | Expiry dates in the register |
| PA-EXC-03 | The inventory lists the exceptions each node carries as `uc_pa_exceptions`, and audits report an excepted clause as excepted, not as passed | 800-53 CA-7 | Audit output |

Development values in the parameters table are not exceptions: the policy itself allows them, and dev nodes are never claimed compliant.

## Deviations from authorities

The authorities disagree in one place. The RHEL 9 STIG requires passwords to expire within 60 days (RHEL-09-411015), while NIST SP 800-63B-4 says verifiers should not require periodic changes. This policy follows the STIG to the letter, and NIST in intent: the controller generates and replaces the passwords itself (PA-PWD-03), so no person has to invent a new password on a schedule, which is the harm NIST's rule exists to prevent. An assessor working strictly to SP 800-63B-4 should be shown this paragraph.

## Key lifecycle example

### Register entries

The key register lives in the controller's inventory. An automation key and an administrator's two FIDO2 keys might read:

```yaml
uc_pa_key_register:
  - id: K-0001
    owner: controller-main
    account: "{{ uc_pa_automation_user }}"
    kind: ssh
    type: ssh-ed25519
    fingerprint: "SHA256:..."
    private_part: controller secret store
    created: 2026-11-20
    expires: 2027-11-20
    status: active
  - id: K-0002
    owner: csteel
    account: admin
    kind: fido2
    type: yubikey-primary
    credential: "pam_u2f mapping for the primary key"
    private_part: on the key
    created: 2026-11-20
    expires: 2027-11-20
    status: active
  - id: K-0003
    owner: csteel
    account: admin
    kind: fido2
    type: yubikey-backup
    credential: "pam_u2f mapping for the backup key"
    private_part: on the key, stored apart from the primary
    created: 2026-12-05
    expires: 2027-12-05
    status: active
```

### Replacing the automation key

1. Generate a new Ed25519 key pair in the controller's secret store and add it to the register as `K-0004`, status `pending`.
2. Run the access play. Every node now holds both `K-0001` and `K-0004` for the automation account.
3. Switch the controller to `K-0004` and run a read-only check against every node. Stop here if any node refuses the new key.
4. Set `K-0001` to `retired` in the register and `K-0004` to `active`.
5. Run the access play again. `K-0001` is removed from every node (PA-KEY-04), and the run fails on any node where it cannot be removed.
6. Destroy the private part of `K-0001` and record the date in the register.

The same steps, with the console in place of SSH, replace an administrator's FIDO2 key. Losing a key starts at step 4 for the lost key, after registering its replacement.

## Changelog

| Version | Status | Notes |
| --- | --- | --- |
| 0.1.0 | Draft | First version, from uc-privileged-access-decision 0.3.0 |

## References

<a id="ref-cccs-nd"></a>
Canadian Centre for Cyber Security. (n.d.). *Identification and authentication* (ITSP.10.033, Security control catalogue). Retrieved November 16, 2026, from https://cyber.gc.ca/en/guidance/cyber-security-privacy-risk-management/itsp10033/identification-authentication
[Return to citation](#cite-cccs-nd)

<a id="ref-cis-2021"></a>
Center for Internet Security. (2021). *CIS Critical Security Controls* (Version 8). https://www.cisecurity.org/controls/v8
[Return to citation](#cite-cis-2021)

<a id="ref-cis-nda"></a>
Center for Internet Security. (n.d.-a). *CIS Debian Linux 12 Benchmark* (Recommendations 5.2.3 and 5.2.4). Retrieved November 16, 2026, from https://workbench.cisecurity.org/benchmarks/18960
[Return to citation](#cite-cis-nda)

<a id="ref-cis-ndb"></a>
Center for Internet Security. (n.d.-b). *CIS Ubuntu Linux 22.04 LTS Benchmark v3.0.0, recommendation 5.2.6: Ensure sudo timestamp_timeout is configured* [Audit item]. Tenable. Retrieved November 16, 2026, from https://www.tenable.com/audits/items/CIS_Ubuntu_Linux_22.04_LTS_v3.0.0_L1_Server.audit:e58d22df6199d2866b61e3c458cc0762
[Return to citation](#cite-cis-ndb)

<a id="ref-disa-2026a"></a>
Defense Information Systems Agency. (2026a). *Canonical Ubuntu 24.04 LTS security technical implementation guide* (Version 1, Release 5). https://dl.dod.cyber.mil/wp-content/uploads/stigs/zip/U_CAN_Ubuntu_24-04_LTS_V1R5_STIG.zip
[Return to citation](#cite-disa-2026a)

<a id="ref-disa-2026b"></a>
Defense Information Systems Agency. (2026b). *Red Hat Enterprise Linux 9 security technical implementation guide* (Version 2, Release 8). https://dl.dod.cyber.mil/wp-content/uploads/stigs/zip/U_RHEL_9_V2R8_STIG.zip
[Return to citation](#cite-disa-2026b)

<a id="ref-disa-2026c"></a>
Defense Information Systems Agency. (2026c). *Red Hat Enterprise Linux 10 security technical implementation guide* (Rule V-281211). STIG Viewer. https://trial.stigviewer.com/stigs/red_hat_enterprise_linux_10/2026-03-11/finding/V-281211
[Return to citation](#cite-disa-2026c)

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
