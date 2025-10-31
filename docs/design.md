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


### Milestone 1 - Run Point Schedules and Property Tests for `cardano-node`

#### Goal

The goal of milestone 1 is to de-risk the project implementation. We will
provide an extremely bare-bones minimum viable product (MVP) which illustrates
that the major engineering hurdles can be solved and that our approach is
feasible.

To that end, we will deliver an executable which can perform black-box test
(a fork of) `cardano-node` against a single point schedule.

The test executable will communicate with `cardano-node` over network sockets,
and will not use any specialized knowledge of the internals of `cardano-node`.
As a possible exception, we will assume that we can run `cardano-node` over
`TestBlock`s. This might require a fork.

We'll start with a simple testing approach, where we compare the resulting
state of the NUT with the end state of the point schedule and make sure
the simulated peers behave as expected.

#### Plan

1. The user obtains a point schedule from somewhere (generating these is part
   of Milestone <TODO>)
2. The user invokes our test binary, passing in the point schedule as an
   argument.
3. The test binary starts up the simulated peers, and returns a topology file.
4. The user starts `cardano-node` with the given topology file.
5. `cardano-node` connects to our simulated peers
6. Once all of the peers have been connected to, the point schedule begins
   running
7. After the point schedule has finished, we **SOME HOW** observe the final
   state of `cardano-node`

   To accomplish this, can we create a new peer, who connects to the NUT, and
   asks to be caught up?
8. The server will exit with a return code corresponding to whether or not
   `cardano-node` ended in the correct state.


### Milestone 2 - Shrinking

#### Goals

Provide a useful failure feedback, so that users can leverage the test
to find specific bugs on their consensus protocol implementation.

Deliver a shrinking strategy to run our executable from [1] on subsequently
smaller tests.

#### Plan

- Change milestone [1] binary to support a shrink index as input (pointing to were
  we currently are on the shrink tree).
    - If test succeeds and there is no shrink index, we end with success.
    - If test succeeds with a shrink index, the binary returns a successor to this
      shrink index.
    - If test fails, we extend the shrink index.
    - If test fails, and the shrink index can't be extended, we output the
      minimally shrunk point schedule.
- Build a utility to materialize the view of the shrinking so that, given some
  point schedule and an index, produces the shrunk point schedule.
  
Alternative: Automatically generate the next shrinking candidate and ask the
client to run it.
We want all our components to be stateless as a design choice for composability.
This work does not preclude the possibility of implementing this.

### Milestone 3 - Amaru and other Implementations

#### Goal

At this point, the MVP is working as expected on the `cardano-node`;
our goal now is to make sure this works for other implementations, and that our design
is effectively decoupled from `cardano-node`. We will figure out the general
requirements for any implementation that adheres to the consensus protocol.

To accomplish the above, in this milestone we make the test work on Amaru
(Rust implementation).

_Stretch:_ Find a test failure in Amaru; which we might not get if its consensus
implementation is correct. But if we find something, this would provide a compelling
argument on the value of this test.

#### Plan

1. Make the test work on Amaru.
    1. Have a means of disabling crypto.
    1. Ensure that it can parse our generated topology files.
1. Document the necessary requirements from a node to run this tests.
    * At the forefront, we have the previous two requirements (parse topology and disable crypto).

#### Questions

Do we need to simulate time? This might be related to configuration access to
node timeouts (as Network delays would be irrelevant in this setting).

### Milestone 4 - Generation of point schedules

#### Goals

Create a separate utility that generates point schedules for specific testing properties.
At this point, we design a file (serialization) format for these.

Note: We will have versioning for the file format, as we work though its constraints.
In the future we might want to generalize this to have a coupling between peer schedules
and the properties being tested.

#### Plan

1. Design a serialization format for point schedules.
1. Produce a binary capable of running different generators and output the serialized
   point schedules.
1. Connect the existing generators (alternatively, expose them).
1. Figure out what other generators we need to write.

### Milestone 5 - UX Improvement

Figure out how to account for the identified common use cases to improve on UX.
How annoying it is to use? What are the crucial pain points?

Clean up, refine and **make it useable**.

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
