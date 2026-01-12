# Conformance Testing of Consensus

Assorted material for design and implementation.

A suite of tools that provide black-box conformance testing for nodes
implementing the Ouroboros Praos consensus protocol. These tools expose part of
[`cardano-node`](https://github.com/IntersectMBO/cardano-node)'s property test
suite to help alternative nodes verify they have implemented the consensus
protocol stack correctly, so that all participating nodes ultimately agree
on the "right" chain when engaging in the protocol.

To accomplish this, upstream peers are simulated to build and serve a
concerted chain, whose generation is informed by possible environmental
conditions (e.g. adversarial behavior and network latency), and the node under
test is judged by its responses to stimuli.

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
