# Interface Contracts

When workstreams have a producer/consumer relationship, define a shared contract
so both agents implement compatible interfaces.

## When to Create a Contract

Create a contract when two workstreams share a boundary where one produces output
the other consumes:

- One workstream exposes an endpoint or module that another calls
- One workstream enqueues jobs, events, or messages that another processes
- Two workstreams share a data shape (e.g., a DTO, payload, or record format)
- A backend workstream returns data that a frontend workstream renders

If the decomposition in Phase 1b identifies a **Consumer/Producer** relationship,
a contract is required.

## Contract Format

Include one contract block per shared interface in the orchestration plan.
Use the project's native type system for the shape definition:

```
Contract: {descriptive name}
Producer workstream: {name}
Consumer workstream: {name}
Direction: {request → response | enqueue → process | emit → handle | render → fetch}

Shape:
  // Define in the project's native type system
  // (TypeScript interface, Python dataclass, Go struct, JSON Schema, etc.)
  //
  // Example:
  // {
  //   id: string
  //   status: "pending" | "processing" | "done" | "failed"
  //   payload: { ... }
  //   createdAt: datetime
  // }

Validation:
  - Producer MUST return/enqueue/emit data matching the Shape exactly
  - Consumer MUST accept/process/handle data matching the Shape exactly
  - Breaking changes require updating BOTH workstreams
```

## How Contracts Flow Through Orchestration

1. **Phase 1b** — The orchestrator identifies Consumer/Producer pairs and defines
   contracts inline in the orchestration plan
2. **Phase 2b** — Each agent's prompt includes the contracts it participates in,
   labeled as PRODUCER or CONSUMER (see the "Interface Contracts" section in the
   agent prompt template)
3. **Phase 4c** — Post-merge verification checks that producer output shapes and
   consumer input shapes still match after integration
4. **Phase 5b-review** — Integration fix review checks that contracts are not
   broken by late-stage changes

## Multiple Contracts

A workstream may participate in multiple contracts with different roles:
- Producer for contract A, consumer for contract B
- Include all applicable contracts in the agent's prompt

## Contract Evolution

If during implementation an agent discovers the contract shape needs to change:
1. The agent MUST note this in the "Integration notes" section of its workstream report
2. The orchestrator updates the contract and re-includes it in the other agent's
   prompt if that agent hasn't completed yet
3. If the other agent has completed, the mismatch will be caught in Phase 4c
   post-merge verification
