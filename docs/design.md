# Conformance Testing of Consensus Design Document

## Motivation

The eponymous feature of a consensus protocol is, of course, that it is expected to maintain
consensus. In our terms, all nodes (subject to the various assumptions on the state of the network
and honest majority) will agree on a prefix of the current chain.
Up to now, this has been achieved in the following ways:

- Every node in the network is running (close to) the same code.
- Consensus testing (living inside `io-sim`, a Haskell IO simulator) validates that all (honest)
nodes will eventually reach consensus.

In a world of multiple node implementations, this strategy no longer holds. Nodes may be running
very different code, and most of it will not be testable under `io-sim`. Nodes failing to agree on the
correct chain risks an accidental hard fork. Should one persist long enough it might potentially be
unrecoverable. Thus, we need to revisit consensus testing in the context of alternative
node implementations. This proposal will extract portions of the Consensus tests of the existing node
that will be of chief benefit to two groups:

- Implementors of alternate nodes, giving them a means of validating that they have
implemented the consensus protocol stack correctly.
- The wider Cardano community, in helping to ensure that the network does not end up
with an accidental hard fork.

## Context

During the course of implementing Ouroboros Genesis, we designed an approach to node
testing which we now call “Node vs Environment”. In effect, while we are ultimately interested in
the behaviour of multiple nodes agreeing on the “right” chain, we can more easily test by taking
advantage of two insights:

1. The logic of identifying the honest chain locally is tricky, but it is very easy to identify
globally. Since we are only interested in cases where there is a global best chain, we
have a very simple judgment rule as to whether a node has selected the correct one.
2. Once we have an easily identified honest chain, we no longer need to simulate multiple
nodes and look for agreement - instead, we simulate a single node and judge the
correctness of its responses to stimuli.

The result of this approach for Ouroboros Genesis was a testing framework that makes use of a
single coordinated point schedule in order to simulate multiple upstream peers (possibly
adversarial, possibly colluding) and validate that a syncing node ends up with the correct chain.
Whilst the point schedule currently is implemented inside the Haskell node, its declarative nature
makes it possible to export this testing method and make it usable across diverse node
implementations.

### Original Proposal

To this end, we aim at the following:

1. Refine the point schedule generators to focus less on syncing nodes (which
   was originally appropriate for Ouroboros Genesis).
2. Define a serialised format for consensus test cases, covering the chain and
   point schedule.
3. Extract from the current node testing suite a standalone capability to
   generate and export point schedules, along with their underlying chains.
4. Create an independent utility that does the following:

   1. Read a serialized point schedule.
   2. Act as one or more peers serving points as defined on the schedule. Such
      peers would instantiate appropriate protocols to serve as upstream peers
      to the node under test (NUT).
   3. Open a timing socket to allow the node under test to control the “ticking”
      of the schedule.
   4. Shrink and restart test cases upon signal from the node under test.
   5. Re-export shrunk test cases to disk — failing test cases that have been
      shrunk are much easier for developers to debug.

This infrastructure would allow other node implementations to take advantage of the work
already done to test the Haskell node, as well as allowing them to validate compatible consensus
behaviour.

## Proposed Change Specification

## Alternatives

## Unresolved Questions

## Implementation Plan

## Milestones

### Milestone 1 - Run Point Schedules and Property tests for the OG implementation

Develop an MVP along the following lines:

1. Build a test binary using a hardcoded point schedule.
    - Alternatively, have the point schedule as input at this point
      so that we can do shrinking.
1. Test binary starts a server and outputs a topology file matching the point
   schedule.
1. Client loads the topology file to connect to our server.
1. Client signals setup completion to start the test.
     - Alternatively, once all the peers have been connected to (or other network
       condition is fulfilled, e.g. a single request message), the test starts
       automatically.
1. Server runs the point schedule, and starts simulating the prescribed peers.
1. Client behaves as normal.
1. Client sends final state back to server.
1. Server checks whether client chose the correct chain.
    - Yes: Success!
    - No: Shrink, rinse and repeat.

### Milestone 2 - Generation of point schedules

1. Parameterize the point schedule.
1. Adapt existing generators.
1. Refine them to our use case (to focus less on syncing nodes).

### Milestone 3 - Shrinking

This is currently the less clear of the milestones, but must definitely be
taken care of at some point:

- Server stays online all during all the testing?
    * Yes: the user only starts the server once and we can keep track of the
      shrink tree in memory.
    * No: We need a separate utility to materialize the view of the shrinking
      process (e.g. to store the current shrink node in disk using an index).
      Then, given some point schedule and an index, produce the shrunk point
      schedule.
- Related: Should we automatically generate the next shrinking candidate and ask
  the client to run it? If so, how?

### Milestone X - All of the above, but for other implementations

## Notes

Other information from the original proposal to take into account during the implementation.

### Dependencies

- The Consensus Team will need to review the Pull Requests that enrich the point schedule
generator (and also add corresponding tests to the existing test suite).
- The architects of alternative nodes would provide very useful feedback at a few phases
throughout the work.

### Maintenance

- As the Consensus algorithm expands—Peras, Leios, etc—it would make sense to enrich
this testing strategy. However, even if that did not happen immediately, maintaining tests
with just a focus on Praos remains important, since Praos is still the foundation that Peras
and Leios enrich.
- It is useful to note that the exact interface of the nodes under test must be relatively
stable, since it’s primarily the same interface that they use to communicate with peers on
mainnet.

### Key Deliverables

- Point schedule generators that are less focused on the syncing node.
- A design for how to manage time in this derived testing setup that no longer benefits from
the io-sim framework.
- The utility for simulating a node’s upstream peers according to the generated point
schedule and also shrinking that point schedule upon failures.
