# uc-contracts

Contracts between independently developed Universal Cake projects. Each contract describes one boundary: what one side hands over, and what the other side may rely on when it receives it.

## Index

| Contract | Version | Status | Egress (producers) | Ingress (consumers) | Document |
| --- | --- | --- | --- | --- | --- |
| `raw-node` | 0.1.0 | Draft | uc-ventoy | uc-controller | [uc-raw-node-contract.md](en/docs/raw-node/uc-raw-node-contract.md) |

## Conventions

### One contract per boundary

A contract is named after what is handed over, such as `raw-node`, not after a project or a direction. The same contract is one project's egress and another's ingress, so it is written once, here.

A contract may have several producers and several consumers. A new producer joins an existing contract rather than creating a new one.

### Layout

```text
uc-contracts/
├── README.md
├── en/
│   └── docs/
│       └── <contract>/
│           └── uc-<contract>-contract.md
└── schema/
    └── <contract>/
```

Each contract has its own folder under `en/docs/`. Machine-readable schemas, when a contract has them, go under `schema/<contract>/`, outside the language root, because they are the same in every language.

### Versions and tags

Each contract has its own version, defined in the contract document. Releases are tagged per contract, for example `raw-node/v0.1.0`, so contracts can change independently of each other.

### Using a contract from another repository

A repository that produces or consumes a contract does not copy it. Instead it lists its contracts in `en/docs/contracts.md`:

| Contract | Version | Role |
| --- | --- | --- |
| `raw-node` | 0.1.0 | Ingress |

The role is `Egress` for a producer and `Ingress` for a consumer. A repository may hold any number of contracts in either role. When it does, it should also appear in the index above.

### When to write a contract

Write a contract when the two sides of a boundary are developed separately and change at different times. Boundaries inside one project, or between parts that always change together, do not need one.
