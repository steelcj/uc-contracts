---
name: uc-onboarding-egress-contract
org: universal-cake
lang: en
type: contract
version: 0.2.1
date: 2026-11-16
---

# Onboarding egress contract

Version: 0.2.1

Status: Draft

Author: Christopher Steel

This contract records what Ansible ingress proves about a node before it hands the node to normalization, and so what normalization may rely on. In short: a node that is ready to be normalized, with a safe connection over SSH and everything the Ansible normalization needs in place beforehand.

It is written from the two ingress playbooks, `00_discover.yml` and `10_onboard.yml`, as supplied on 2026-11-16. The roles those playbooks call were not read. Where a clause depends on what a role does, it says only what the playbook and its comments say.

## Stages

| Stage | Playbook | What it does | Changes the node | Covered here |
| --- | --- | --- | --- | --- |
| Discovery | `00_discover.yml` | Finds raw nodes and writes them to a generated inventory | No | Only as the source of the node's provisional name |
| Onboarding | `10_onboard.yml` | Makes a raw node manageable | Yes | Yes, as what it proves |
| Normalization | `20_normalize.yml` | Brings the node to its managed state | Yes | Yes, as what it may rely on |

## Scope

Every clause below describes the node, or the controller's record of it, when `10_onboard.yml` has finished for that node. Normalization may rely on every clause.

The contract does not say what happens when a clause fails. In the playbook, a failed task stops onboarding for that node before it is written to the onboard inventory.

## How a clause is proved

| Proof | Meaning |
| --- | --- |
| Tested | A task at the end of the playbook checks it on the node, after every change has been made |
| Converged | A role or task that establishes it ran without failing. Nothing checks the result separately. |
| Not yet | Intended, but the playbook as supplied does not deliver it |

## What normalization receives

The last plays re-target the node and write it to the onboard inventory. Normalization reaches the node with these values:

| Value | Set to |
| --- | --- |
| Group | `onboarded`, and `target_group` when one is given |
| Connection | SSH |
| User | `initial` |
| Key | `onboard_ssh_private_key_file`, and only that key is offered |
| Address | The node's default IPv4 address from facts gathered during onboarding, falling back to the inventory address, then the inventory name |

## Clauses

### Connection

| ID | The onboarded node provides | Proof |
| --- | --- | --- |
| OE-CON-01 | The node answers over SSH as `initial`, with the key in `onboard_ssh_private_key_file` and no other key offered | Tested, by `wait_for_connection` in the play "Confirm SSH as the initial user works" |
| OE-CON-02 | `initial` obtains root through sudo without a password | Tested, by running `id` with become in the same play |
| OE-CON-03 | A Python interpreter that Ansible can use is present | Tested, because the two tasks above cannot run without it |
| OE-CON-04 | The firewall and fail2ban let the controller in over SSH | Tested, because OE-CON-01 runs after both were applied |

### Accounts

| ID | The onboarded node provides | Proof |
| --- | --- | --- |
| OE-ACC-01 | The account `initial` exists, with its sudo rule and its public key | Converged, by `robertdebock.users`. Its effect is then tested by OE-CON-01 and OE-CON-02. |
| OE-ACC-02 | sudo is installed | Converged, by `robertdebock.bootstrap` |

### Baseline

In the order the playbook applies them.

| ID | The onboarded node provides | Proof |
| --- | --- | --- |
| OE-BAS-01 | A tight system-wide umask, set before the `initial` account is created. The values come from the onboard inventory. | Converged, by `steelcj.umask` |
| OE-BAS-02 | The EPEL repository, where the role applies | Converged, by `robertdebock.epel` |
| OE-BAS-03 | Packages are updated | Converged, by `robertdebock.update` |
| OE-BAS-04 | The locale is set | Converged, by `robertdebock.locale` |
| OE-BAS-05 | Automatic updates are configured | Converged, by `robertdebock.auto_update` |
| OE-BAS-06 | cron is present | Converged, by `robertdebock.cron` |
| OE-BAS-07 | Time is synchronised, except in containers, which take their clock from the host | Converged, by `robertdebock.ntp`. The playbook notes that Debian 13 dropped the `ntp` package and that this needs rethinking. |

### SSH

| ID | The onboarded node provides | Proof |
| --- | --- | --- |
| OE-SSH-01 | The SSH server is installed and hardened | Converged, by `robertdebock.openssh`. Which settings it hardens was not read. |
| OE-SSH-02 | fail2ban is in place | Converged, by `robertdebock.fail2ban` |

### Firewall

| ID | The onboarded node provides | Proof |
| --- | --- | --- |
| OE-FW-06 | A firewall is applied from `firewall_rules_onboard`, and those rules include SSH | Converged, by `uc_firewall`, which refuses to run without the SSH rule |

The clauses below were drafted in version 0.1.0 from `ansible-role-foomuuri`. The owner has decided that `ansible-role-foomuuri` will replace `uc_firewall` in `10_onboard.yml`. The playbook as supplied still runs `uc_firewall`, so none of these clauses is delivered yet. When the replacement is made they take over from OE-FW-06, which is then retired.

| ID | The onboarded node provides | Proof |
| --- | --- | --- |
| OE-FW-01 | nftables is the only packet filter: there are no iptables-legacy rules, and no other firewall manager is active | Not yet |
| OE-FW-02 | One firewall manager owns one nftables table, and nothing flushes the whole ruleset at boot or when a service stops | Not yet |
| OE-FW-03 | Inbound, only SSH, DHCP and DHCPv6 replies, ping, and the ICMP and ICMPv6 that networking needs are accepted. Outbound, everything is accepted. Forwarded, nothing is accepted. | Not yet |
| OE-FW-04 | The firewall is in force before the network comes up at boot | Not yet |
| OE-FW-05 | Every other firewall service installed on the node is stopped, disabled and masked | Not yet |
| OE-FWM-01 | The firewall manager is Foomuuri | Not yet |
| OE-FWM-02 | Foomuuri's configuration is only the fragment `/etc/foomuuri/ansible/10-base.conf` | Not yet |
| OE-FWM-03 | Foomuuri's D-Bus handler, `foomuuri-dbus.service`, is masked and not running | Not yet |

### Inventory

| ID | The controller holds | Proof |
| --- | --- | --- |
| OE-INV-01 | The node is written to the onboard inventory with its address and key file, by `promote_node.py`. The change is not committed. | Converged, by the last play |

Identifiers are never reused. A removed clause keeps its identifier retired.

## Order of proof

The playbook's order carries meaning, and its comments say why:

1. **Python and sudo first.** `robertdebock.bootstrap` installs both, because nothing else can run without Python and the users role needs sudo.
2. **Facts are gathered fresh**, never from the cache, because the address used for the SSH switch comes from them.
3. **The umask is set before any user is created**, so that the home directory of `initial` is created with the intended permissions.
4. **The SSH server, fail2ban and the firewall come before the user.**
5. **The `initial` account is created last.**
6. **Only then does the connection switch** from root to SSH as `initial`, and the tests in Connection run over that new connection.

## Overlap with the raw node contract

The playbook and the raw node contract describe a raw node differently. Each row is a point where onboarding repeats, changes or contradicts what the contract says the node already has.

The owner has decided that the raw node contract is the current description: a raw node is reached as the initial user with sudo, and has a firewall that allows SSH. The playbook as supplied predates that and is to be brought in line. Until it is, the rows below stand.

| Subject | Raw node contract | `10_onboard.yml` | Effect |
| --- | --- | --- | --- |
| How the node is reached | SSH as `initial_username` with a key. `root` cannot log in over SSH (RN-ACC-02, RN-ACC-05). | As root, over the Incus connection or over SSH | A node that meets the contract cannot be reached as root over SSH |
| The initial account | Exists at handoff (RN-ACC-01) | Created last. Its existence is the test for "already onboarded", and such a node is skipped. | If `initial_username` is `initial`, a node that meets the contract is skipped and never onboarded |
| umask and the home directory | The home directory exists at handoff | The umask is set first so the home directory is created under it | On a node that meets the contract the home directory already exists, so the umask does not shape it |
| sudo | With a password (RN-ACC-04) | Without a password, and tested that way | Onboarding changes it |
| Python | Reported, not promised | Installed by bootstrap | Consistent. Onboarding supplies what the contract leaves out. |
| SSH server | Listening, with key login and password login as a recovery path (RN-SSH-01, RN-ACC-02, RN-ACC-03) | Installed and hardened | Repeated. Whether password login survives depends on the role's settings, which were not read. |
| Firewall | A firewall that meets RN-FW-01 to RN-FW-04. No manager is named, since RN-FWM-01 was retired in raw node contract 0.2.0. | `uc_firewall`, from `firewall_rules_onboard` | Reconciled once `ansible-role-foomuuri` replaces `uc_firewall`. The role assumes that a Foomuuri-managed firewall is wanted, and nothing about the firewall the node arrives with. It installs Foomuuri over whatever is there. |
| Checking the node | Ingress checks the node against the contract, reading its confirmation file | Discovery scans a subnet for hosts. Nothing reads the confirmation file. | The contract check is not in the playbooks supplied |
| Name | A hostname that is set (RN-NET-01) | A provisional inventory name, `disc-<ip>`. Renaming is not done here. | Not reconciled |

## Not guaranteed

Normalization must not rely on these:

- any inbound port other than SSH
- the node's final name
- time synchronisation in a container, or on Debian 13 until the `ntp` question is settled
- a committed inventory. The promotion is written to the working tree and left for the operator to commit.
- anything a role does beyond what its line in the playbook says

## Open points

- When `10_onboard.yml` is brought in line with the raw node contract, which clauses change? Connection, Accounts and Order of proof all describe a node that is first reached as root and has no `initial` account yet.
- Should the Converged clauses gain a test at the end of the playbook, so that normalization relies on proof and not on a role having run?
- Should onboarding write a report of its own, as the raw node writes its confirmation file?

## Versioning

The contract has its own version, independent of the raw node contract.

| Change | Version part |
| --- | --- |
| A clause is removed, weakened or changed in meaning, or the firewall manager in OE-FWM-01 changes | Major |
| A clause is added | Minor |
| Wording is clarified without changing what either side may rely on | Patch |

While the major version is 0, a minor version may also be incompatible, and the changelog says when it is.

## Changelog

| Version | Status | Notes |
| --- | --- | --- |
| 0.2.1 | Draft | Two decisions by the owner recorded, with no clause changed: `ansible-role-foomuuri` will replace `uc_firewall`, and the raw node contract is the current description of a raw node. Two open points removed and one added. |
| 0.2.0 | Draft | Rewritten from the ingress playbooks: clauses for connection, accounts, baseline, SSH, firewall and inventory, each with how it is proved. Order of proof and overlap with the raw node contract added. The version 0.1.0 firewall clauses are kept and marked not yet delivered. Incompatible with 0.1.0. |
| 0.1.0 | Draft | First draft: firewall and firewall manager clauses. SSH, normalization prerequisites and the report left open, with candidates. |
