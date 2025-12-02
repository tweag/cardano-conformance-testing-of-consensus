# Project Notes

A collection of questions and notes from elsewhere.

## How to inject simulated peers into an alternate node for conformance testing?

The basic idea is leveraging the existing diffusion layer from
`ouroboros-network` and a lightweight example like `immdb-server`.

### High-level overview

- In the current Genesis tests, clients and servers are plugged together purely
  in-memory (`createConnectedChannels` in `PeerSimulator.Run`). This doesn't
  involve any serialization.
- For conformance testing, we instead want the servers to be accessible over a
  regular network connection, ie exposing a port (or more than one), speaking
  the same network protocol as regular nodes. The node that is going to be
  tested is then configured to only connect to such peers. For this, the
  existing diffusion layer from `ouroboros-network` can be used.
  - An existing minimal example (ie something that is much more lightweight than
    a full node, but still can speak the server-side mini-protocols) is
    [`immdb-server`](https://github.com/IntersectMBO/ouroboros-consensus/tree/main/ouroboros-consensus-cardano/src/unstable-cardano-tools/Cardano/Tools/ImmDBServer)
    ([brief docs](https://github.com/IntersectMBO/ouroboros-consensus/tree/main/ouroboros-consensus-cardano#immdb-server)).
  - It only uses a simplified version of the diffusion layer, but this seems
    fine for now, and maybe even is sufficient for the final product.
  - Instead of implementing ChainSync/BlockFetch servers in terms of reading
    from an ImmutableDB, the tool you are writing would instead defer to
    serving a peer schedule.

One non-trivial aspect here (just a brief sketch, let me know if this is too
vague): The Genesis tests are currently written against `Test.Util.TestBlock`
(although some parts are block-generic). Standardizing a new kind of test block
is significant work, and the big downside is that other node implementations
would need to implement it in order to benefit from the testing framework.

Therefore, it seems sensible want to use actual Cardano blocks, as this is
what other nodes actually implement. However, this is also non-trivial, as
Cardano blocks are more cumbersome to work with (for one part, because they
involve actual cryptography). Most crucially, the leader schedule is governed
by a VRF for Cardano, so we can't just "generate" a leader schedule as we like
(without further work such as disabling the VRF checks just for conformance
testing).

## What interface demands are we making of alternative nodes?

The node that is going to be tested needs to be configured to only connect to
the simulated peers as `localRoots`. For this, `cardano-node` uses a
[topology file](https://developers.cardano.org/docs/operate-a-stake-pool/node-operations/topology/)
which is currently not a standard, but at least Amaru also parses this part
of it.

One other idea alluded to in the [proposal](proposal.md) text would be some
custom interface to allow the node under test to signal that it is ready for
the next tick:

> Open a timing socket to allow the node under test to control the “ticking”
> of the schedule.

But this still needs a concrete design, and doesn't seem required for an
initial prototype.
