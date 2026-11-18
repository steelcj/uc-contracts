# uc-contracts

Contracts, policies and decisions shared by independently developed Universal Cake projects.

- A contract describes one boundary: what one side hands over, and what the other side may rely on when it receives it.
- A policy describes what must always be true of something, such as a normalized node, whichever project produced or manages it.
- A decision records why a contract or policy says what it does, and which options were rejected.
- adding a line to the raw node contract that `mac` and `machine_id` are quoted strings, so every future producer does the same.

## Index

### Contracts

| Contract | Version | Status | Egress (producers) | Ingress (consumers) | Document |
| --- | --- | --- | --- | --- | --- |
| `raw-node` | 0.3.0 | Draft | uc-ventoy | uc-controller | [uc-raw-node-contract.md](en/docs/node-onboarding-pipeline/1-raw-node-egress/uc-raw-node-contract.md) |
| `onboarding-egress` | 0.3.0 | Draft | uc-controller (onboarding) | uc-controller (normalization) | [uc-onboarding-egress-contract.md](en/docs/node-onboarding-pipeline/2-ansible-ingress-egress-contract/uc-onboarding-egress-contract.md) |

### Policies

| Policy | Version | Status | Applies to | Document |
| --- | --- | --- | --- | --- |
| `privileged-access` | 0.1.0 | Draft | Normalized nodes | [uc-privileged-access-policy.md](en/docs/node-policies/privileged-access/uc-privileged-access-policy.md) |
| Exceptions register | 0.1.0 | Current | Every node policy | [uc-policy-exceptions.md](en/docs/node-policies/exceptions/uc-policy-exceptions.md) |

### Decisions

| Decision | Version | Status | Document |
| --- | --- | --- | --- |
| Privileged access for raw and onboarded nodes | 0.3.0 | Proposed | [uc-privileged-access-decision-v0.3.0.md](en/docs/decisions/uc-privileged-access-decision-v0.3.0.md) |

## Conventions

### One contract per boundary

A contract is named after what is handed over, such as `raw-node`, not after a project or a direction. The same contract is one project's egress and another's ingress, so it is written once, here.

A contract may have several producers and several consumers. A new producer joins an existing contract rather than creating a new one.

### Policies and contracts

A policy states what must hold for the whole life of what it covers. Contracts apply the parts of a policy that the thing handed over can already meet at that boundary, and say so in their text. A policy clause keeps its identifier even when a contract enforces it under an identifier of its own.

### Layout

```text
uc-contracts/
├── README.md
├── en/
│   └── docs/
│       ├── decisions/
│       │   ├── artifacts/
│       │   │   └── uc-<subject>-decision-v<earlier>.md
│       │   └── uc-<subject>-decision-v<version>.md
│       ├── node-onboarding-pipeline/
│       │   └── <stage>-<boundary>/
│       │       └── uc-<contract>-contract.md
│       └── node-policies/
│           ├── exceptions/
│           │   └── uc-policy-exceptions.md
│           └── <policy>/
│               └── uc-<policy>-policy.md
└── schema/
    └── <contract>/
```

A decision's file name carries its version, and earlier versions move to `decisions/artifacts/`. Contracts that belong to one pipeline sit together in a folder for that pipeline, one folder per boundary, numbered in the order a node meets them. Machine-readable schemas, when a contract has them, go under `schema/<contract>/`, outside the language root, because they are the same in every language.

### Versions and tags

Each contract, policy and decision has its own version, defined in its document. Releases are tagged per document, for example `raw-node/v0.3.0` or `privileged-access/v0.1.0`, so they can change independently of each other.

### Using a contract or policy from another repository

A repository that produces or consumes a contract, or implements a policy, does not copy it. Instead it lists them in `en/docs/contracts.md`:

| Document | Version | Role |
| --- | --- | --- |
| `raw-node` | 0.3.0 | Ingress |
| `privileged-access` | 0.1.0 | Implements |

The role is `Egress` for a producer, `Ingress` for a consumer, and `Implements` for a repository that enforces a policy. A repository may hold any number of them. When it does, it should also appear in the index above.

### When to write a contract

Write a contract when the two sides of a boundary are developed separately and change at different times. Boundaries inside one project, or between parts that always change together, do not need one.

### When to write a policy

Write a policy when a requirement must hold across several stages or projects, so that writing it into one contract would leave the others free to undo it.
