# Conformance Testing of Consensus

A project approved within
[TWEAG’s Proposals for multiple core budget projects for Cardano 2025](https://governancespace.com/en-us/budget-discussions/205).

This repository is intended for project tracking and for documenting various
aspects of the design and implementation.

A suite of tools that provide black-box conformance testing for nodes
implementing the Ouroboros Praos consensus protocol. These tools expose
[`cardano-node`](https://github.com/IntersectMBO/cardano-node)'s
*node vs environment* tests to help alternative nodes verify they have
implemented the consensus protocol stack correctly, so that all participating
nodes ultimately agree on the "right" chain when engaging in the protocol.

To accomplish this, upstream peers are simulated to build and serve a
concerted chain, whose generation can be tuned to simulate extreme,
but possible, environmental conditions caused, for example, by coordinated
adversarial behavior and network latency). The node under test is judged by
its responses to stimuli.

See the [design document](./docs/design.md) for details.

## Development

Ongoing development is carried out in the following Cardano package forks,
on the corresponding `conformance-testing` branches (branched from package
versions matching the`cardano-node`
[10.5.1 release](https://github.com/IntersectMBO/cardano-node/releases/tag/10.5.1)):

- [cardano-node](https://github.com/tweag/cardano-node/tree/conformance-testing),
contains the following executables:
  - [`conformance-test-runner`](https://github.com/tweag/cardano-node/blob/conformance-testing/cardano-node/app/conformance-test-runner.hs)
  - [`conformance-test-viewer`](https://github.com/tweag/cardano-node/blob/conformance-testing/cardano-node/app/conformance-test-runner.hs)

- Patching of [ouroboros-consensus](https://github.com/tweag/ouroboros-consensus-testing/tree/conformance-testing)
to expose the property testing infrastructure.
