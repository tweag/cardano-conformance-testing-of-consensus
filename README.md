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
