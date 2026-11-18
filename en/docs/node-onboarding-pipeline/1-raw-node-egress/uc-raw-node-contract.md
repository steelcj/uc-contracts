---
name: uc-raw-node-contract
org: universal-cake
lang: en
type: contract
version: 0.3.1
date: 2026-11-17
---

# Raw node contract

Version: 0.3.1

Status: Draft

Author: Christopher Steel

This contract defines the state of a raw node at the moment a producer hands it over, and what Ansible ingress may rely on when it receives the node. It is the agreement between raw node producers (egress) and the Ansible controller (ingress). It does not describe how a producer fulfils it, or what happens after ingress.

The [privileged access policy](../../node-policies/privileged-access/uc-privileged-access-policy.md) governs normalized nodes. This contract applies the parts of it that a raw node can already meet, for the reasons recorded in [uc-privileged-access-decision](../../decisions/uc-privileged-access-decision-v0.3.0.md).

## Stages

| Stage | What it does | Changes the node | Covered here |
| --- | --- | --- | --- |
| Egress | A producer builds a raw node | Yes | Yes, as obligations |
| Ingress | Discovery finds the node and checks it against this contract | No, read-only | Yes, as what it may rely on |
| Onboarding | Prepares the node for normalization, and handles nodes that failed the contract | Yes | No |
| Normalization | Brings the node to its managed state | Yes | No |

## Scope

Every clause below describes the raw node as it is when the producer hands it over. Ingress may rely on every clause and may check any of them without changing the node. The contract does not say how a producer meets a clause, and it does not say what happens when a clause fails.

The contract is the same for every Linux distribution. Which producers and targets implement it is recorded outside this document, by each producer.

## Shared parameters

Both sides know these values for each node:

| Parameter | Meaning |
| --- | --- |
| `initial_username` | The temporary account used for ingress |
| `initial_password` | The password of `initial_username`. Ingress holds it in the controller's secret store, and the producer receives only its hash. |
| `ssh_public_key` | The public key whose private key ingress holds |
| `ssh_port` | The SSH port. Default 22. |
| `ssh_sources` | The addresses or networks allowed to reach SSH. Empty means any. |

A producer builds nodes before their hostnames exist, so `initial_password` may be shared by every node of one build batch. Onboarding replaces it with a password unique to the node.

Ingress holds `initial_password`. Every other secret used to build the node, such as the `root` password, belongs to the producer, and ingress never needs it.

## Clauses

### Network

| ID | The raw node provides |
| --- | --- |
| RN-NET-01 | A hostname that is set and is not the installer's default |

### Accounts

| ID | The raw node provides |
| --- | --- |
| RN-ACC-01 | The account `initial_username` exists, and its home directory holds only `.ssh/authorized_keys` and the distribution's defaults for a new home |
| RN-ACC-02 | `initial_username` can log in over SSH with `ssh_public_key` |
| RN-ACC-03 | Retired in 0.3.0. Was: `initial_username` can log in over SSH with its password, as a recovery path. Replaced by RN-ACC-06 and RN-SSH-03. |
| RN-ACC-04 | `initial_username` can obtain root privileges through sudo, with its password |
| RN-ACC-05 | The `root` account has a password and cannot log in over SSH |
| RN-ACC-06 | `initial_username` can log in at the console with its password, as the recovery path during the raw window |
| RN-ACC-07 | No sudoers rule contains `NOPASSWD` or `!authenticate`. A comment line is not a rule. |
| RN-ACC-08 | `initial_password` is at least 15 characters |

### SSH

| ID | The raw node provides |
| --- | --- |
| RN-SSH-01 | The SSH server listens on `ssh_port` |
| RN-SSH-02 | SSH is reachable from `ssh_sources`, or from anywhere when the list is empty, on at least one address reported in `interfaces` |
| RN-SSH-03 | The SSH server accepts public key authentication only. Password, keyboard-interactive and empty-password logins are off. |

### Firewall

The raw node has a firewall that accepts SSH on `ssh_port` from `ssh_sources`, and little else. The clauses are stated in nftables terms, so that they hold whichever firewall manager the producer uses.

| ID | The raw node provides |
| --- | --- |
| RN-FW-01 | nftables is the only packet filter: there are no iptables-legacy rules, and no other firewall manager is active |
| RN-FW-02 | One firewall manager owns one nftables table, and nothing flushes the whole ruleset at boot or when a service stops |
| RN-FW-03 | Inbound, only SSH as in RN-SSH-02, DHCP and DHCPv6 replies, ping, and the ICMP and ICMPv6 that networking needs are accepted. Outbound, everything is accepted. Forwarded, nothing is accepted. |
| RN-FW-04 | The firewall is in force before the network comes up at boot |

### Firewall manager

No clauses. The firewall manager is the producer's choice, and the confirmation reports which one is active in `firewall`. The firewall clauses above are all a raw node promises about its firewall. Onboarding may keep the manager it finds or replace it with its own, and ingress must not rely on any particular one.

| ID | Status |
| --- | --- |
| RN-FWM-01 | Retired in version 0.2.0. It said that the firewall manager is Foomuuri, with its configuration only in the file `/etc/foomuuri/00-raw-node.conf`. |

### Report

| ID | The raw node provides |
| --- | --- |
| RN-REP-01 | The file `/var/lib/raw-node/confirmation.yml` exists, is readable by `initial_username`, and holds the fields defined below, measured on the node |

Identifiers are never reused. A removed clause keeps its identifier retired, so a report naming a clause always means the same thing.

## Confirmation

### Format

The confirmation is YAML, keyed by the node's hostname:

```yaml
deb13-vt-a1b2:
  contract: "0.3.1"
  baseline: "0.4.0"
  timestamp: "2026-11-15T19:35:00-05:00"
  machine_id: "4c1f0a9e2b7d4e8f9a0b1c2d3e4f5a6b"
  user: "initial"
  sudo: true
  sudo_nopasswd: false
  ssh_port: 22
  ssh_password_auth: false
  ssh_host_keys:
    - type: "ssh-ed25519"
      fingerprint: "SHA256:k3J8m0yQvXb2..."
    - type: "ecdsa-sha2-nistp256"
      fingerprint: "SHA256:Qe1pT9wZr4Lc..."
  firewall: "foomuuri"
  firewall_active: true
  python: "/usr/bin/python3"
  interfaces:
    - name: "enp0s31f6"
      type: "ethernet"
      mac: "8c:8c:aa:11:22:33"
      state: "up"
      default_route4: true
      default_route6: true
      ip: ["192.0.2.10"]
      ip6: ["2001:db8::10"]
    - name: "wlp0s20f3"
      type: "wireless"
      mac: "70:cf:49:44:55:66"
      state: "down"
      default_route4: false
      default_route6: false
      ip: []
      ip6: []
```

### Value types

Only the booleans (`sudo`, `sudo_nopasswd`, `ssh_password_auth`, `firewall_active`, `default_route4`, `default_route6`) and `ssh_port` are anything other than strings. Every other value is a string, including empty values, which are written `""`, and every item in the `ip` and `ip6` lists.

YAML 1.1 parsers, which include PyYAML and therefore Ansible, otherwise change some values silently:

| Value | Unquoted, read as |
| --- | --- |
| `machine_id` made only of decimal digits | An integer, losing any leading zeros |
| `mac` made only of decimal digits and colons, such as `52:54:00:12:34:56` | A base 60 integer |
| `timestamp` | A datetime, not the text that was measured |
| An IPv6 address made only of decimal digits and colons | A base 60 integer |
| `contract` or `baseline`, such as `1.0` | A number |

So:

- A producer should write every string value in double quotes, as in the example above. From the next minor version this is required.
- A consumer must read every value as a string, for example with PyYAML's `BaseLoader`, and convert the booleans and `ssh_port` itself. It must not rely on a producer quoting.

### Fields

Every value is measured on the node when the report runs, not copied from the producer's input.

| Field | Meaning |
| --- | --- |
| Top-level key | The node's hostname |
| `contract` | The version of this contract the node implements |
| `baseline` | The producer's own baseline version |
| `timestamp` | When the report ran, in ISO 8601 with offset |
| `machine_id` | The contents of `/etc/machine-id`. Identifies this installation, and changes on reinstall. |
| `user` | `initial_username` if that account exists, otherwise empty |
| `sudo` | Whether `initial_username` can use sudo |
| `sudo_nopasswd` | Whether any sudoers rule contains `NOPASSWD` or `!authenticate`. Must be `false` (RN-ACC-07). |
| `ssh_port` | The port the SSH server listens on |
| `ssh_password_auth` | Whether the running SSH server accepts password or keyboard-interactive authentication, as reported by `sshd -T`. Must be `false` (RN-SSH-03). |
| `ssh_host_keys` | Every SSH host key of the node, as a list of `type` and SHA256 `fingerprint` |
| `ssh_host_keys[].type` | The key type, such as `ssh-ed25519` |
| `ssh_host_keys[].fingerprint` | The key's SHA256 fingerprint, as printed by `ssh-keygen -l` |
| `firewall` | The name of the active firewall manager |
| `firewall_active` | Whether the firewall manager is running |
| `python` | The path of the system Python 3 interpreter, or empty when there is none. Reported, not promised. |
| `interfaces` | Every network interface except loopback, as found |
| `interfaces[].name` | The kernel name of the interface |
| `interfaces[].type` | `ethernet`, `wireless` or `other` |
| `interfaces[].mac` | The permanent hardware address, not one that may be randomised |
| `interfaces[].state` | `up` or `down` |
| `interfaces[].default_route4` | Whether the interface holds an IPv4 default route |
| `interfaces[].default_route6` | Whether the interface holds an IPv6 default route |
| `interfaces[].ip` | The IPv4 addresses on the interface, an empty list when there are none |
| `interfaces[].ip6` | The global IPv6 addresses on the interface, an empty list when there are none |

The network interface fields are facts only. The contract makes no promise about how many interfaces a node has, which of them are up, or which holds a default route.

### Identity

`machine_id` and the interfaces' MAC addresses together let ingress match a node with what it already knows:

| `machine_id` | MAC addresses | Reading |
| --- | --- | --- |
| Known | Known | The same installation on the same hardware |
| Known | New | The same installation, on other hardware or with other network devices |
| New | Known | The same hardware, reinstalled |
| New | New | A new node |

Anyone with root on a node can change both values. Ingress may use them to match and track nodes, but never to decide whether to trust one.

### Host keys

`ssh_host_keys` is what ingress trusts. Ingress compares the fingerprints in the confirmation with those of the host keys the node presented on the connection that read it, and admits the node only when they match. The confirmation is most useful when it is also read out of band, such as from the console, so that the comparison does not rely on the same connection alone.

## Not guaranteed

The contract says nothing about these, and ingress must not rely on them:

- package versions, update state, and whether automatic updates run
- accounts other than `initial_username` and `root`, and services other than SSH and the firewall
- disk layout
- locale, time zone and the clock
- Python
- anything about network interfaces beyond what the confirmation reports

## Versioning

The contract has its own version, independent of any producer's baseline. Every node records the version it implements in the confirmation's `contract` field.

| Change | Version part |
| --- | --- |
| A clause is removed, weakened or changed in meaning, or a confirmation field is removed, renamed or changes meaning | Major |
| A clause or a confirmation field is added | Minor |
| Wording is clarified without changing what either side may rely on | Patch |

While the major version is 0, a minor version may also be incompatible, and the changelog says when it is.

## Changelog

| Version | Status | Notes |
| --- | --- | --- |
| 0.3.1 | Draft | Wording only, no clause or field changed: the unfinished firewall text replaced by a sentence in terms of `ssh_port` and `ssh_sources`; the firewall manager stated to be the producer's choice, which onboarding may keep or replace, correcting the statement that Foomuuri is installed during onboarding; RN-ACC-07 clarified, a comment line is not a rule; a section on value types added, with producers advised to quote every string value and consumers required to read every value as a string; the example quoted accordingly; a stray draft note removed |
| 0.3.0 | Draft | Incompatible with 0.2.0. Applies the privileged access policy where a raw node can meet it: shared parameter `initial_password` added; RN-ACC-03 (password login over SSH) retired; RN-ACC-06 (console login with the password), RN-ACC-07 (no `NOPASSWD`), RN-ACC-08 (password length) and RN-SSH-03 (key-only SSH) added; confirmation fields `sudo_nopasswd`, `ssh_password_auth` and `ssh_host_keys` added, with how ingress uses host keys |
| 0.2.0 | Draft | RN-FWM-01 retired: the contract no longer names a firewall manager, because Foomuuri is installed during onboarding, after this contract. Incompatible with 0.1.0 for anything that relied on a raw node running Foomuuri. |
| 0.1.0 | Draft | First version, agreed in discussion: clauses for hostname, accounts, SSH, the nftables firewall and its manager, and the confirmation report with interfaces and identity |
