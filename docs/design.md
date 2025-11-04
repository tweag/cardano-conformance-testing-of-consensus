# Conformance Testing of Consensus Design Document

## Motivation

`cardano-node` ships with a large suite of tests verifying its consensus
protocol. Given the subtlety in the implementation of the consensus protocol,
it's desirable that we can leverage this existing test suite to help verify the
correctness of alternative node implementations.

Correctness here is an *extremely important* property---much more so than in
most software projects. Nodes failing to agree on the correct chain risks an
accidental hard fork. Should one persist long enough it might potentially be
unrecoverable.

This document gives a design for a suite of tools, and the necessary
infrastructure changes, to expose these existing tests in a form that
alternative nodes can use.

We do not make any assumptions that alternative nodes be written in Haskell,
nor that they have access to a QuickCheck-like library.


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

### Design Desiderata

- composable
- reusable
- stateless
- useful


### Design

We will ship three separate tools:

1. Test Generator (testgen)
2. Test Runner (runner)
3. Shrink Viewer (shrinkview)

The purpose of `testgen` is to generate test cases. Testgen will be a CLI tool
which accepts arguments to select a specific class of tests, and potentially
some test-specific tuning knobs (to eg, change the "difficulty" of the test.)
Each class of tests will have an associated `Gen`erator, which testgen will
invoke to instantiate a specific instance of the test class. Testgen will
output a test file, the contents of which will contain a point schedule and
(mechanical) description of the property which needs to pass.

<!-- TODO(sandy): how do we serialize properties? -->

The `runner` CLI tool accepts a test file (as output by `testgen`) and
a *shrink index* (see @sec:shrinking), and spins up simulated peers
corresponding to the embedded point schedule. The `runner` tool will then
output a [topology
file](https://developers.cardano.org/docs/operate-a-stake-pool/node-operations/topology/)
whose `localRoots` will point to the simulated peers. We will create an
additional `localRoot` peer whose job is to record all messages diffused from
the NUT.

<!-- TODO(sandy): define NUT sooner -->

Alterantive nodes which wish to test against `runner` (the "nodes under test",
or *NUTs*) can parse the generated topology file and connect to the simulated
peers. Once they have all been connected to, the point schedule will begin
running. The simulated peers will follow the point schedule, sending their
mocked blocks to the NUT.

Upon completion of the point schedule, we will evaluate the test property. We
can compare the final state of the NUT (as observed by the testing peer) and
ensure the desired property holds. Depending on the result of the test and the
state of the shrink index, we will perform different actions (see @sec:exit-codes.)

The `shrinkView` tool accepts a test file and a shrink index, and outputs the
test file corresponding to the given shrink index. This tool is primarily
useful for looking at non-minimal test inputs, eg, when the user doesn't want
to iterate the shrinking all the way down to a minimal example.


### Exit Codes

The `Exit` bit mask enum is used in the following section:

```c
enum Exit {
  SUCCESS = 0,
  INTERNAL_ERROR = 1,
  BAD_USAGE = 2,
  TEST_FAILED = 4,
  CONTINUE_SHRINKING = 8,
}
```

In the case that the property succeeded and the shrink index is `empty`, we
will exit with code `SUCCESS`. This corresponds to a test pass.

`INTERNAL_ERROR` is for when something goes wrong inside of `runner` itself,
and `BAD_USAGE` is for when the program is invoked incorrectly (eg called with
unparsable flags.) This usage of codes 1 and 2 is consistent with POSIX
standards.

If the property succeeded, but the shrink index was non-`empty`, we will exit
with code `CONTINUE_SHRINKING`. In addition, we will output the result of `succ
shrinkIndex` on stdout. While this is technically a test pass, it is a pass for
a shrunk input. Thus this is merely a "local" success, rather than a "global"
success.

If the property failed, and we can `extend` the shrink index, we will exit with
code `TEST_FAILED | CONTINUE_SHRINKING` and output the result of `extend shrinkInput`
on stdout. This corresponds to a non-minimal test failure.

If the property failed and we cannot `extend` the shrink index, we will exit
with code `TEST_FAILED`, and produce the minimal test case on stdout.

When the `CONTINUE_SHRINKING` bit is part of the exit code, the user is
encouraged to restart the `runner` with the new shrink index, in order to
manually "pump" the shrinker.


## Alternatives

## Unresolved Questions

* Do we need a separate peer to act as our state observer? Maybe not, but it's
  conceptually clearer to have a peer whose sole job is to collect data. As
  a counterexample, what happens when the peer schedule is empty?

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
