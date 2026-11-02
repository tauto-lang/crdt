# crdt

Conflict-free replicated data types (CRDTs) for [Tauto](https://github.com/tauto-lang), with proofs. Replicas can exchange states in any order, with duplicates, and still end up in the same state. The proofs are checked by the Tauto kernel, and the code you import is the code that was proved. It depends only on std.

State-based and delta-state CRDTs are included:

- counters: grow-only (G-Counter) and increment/decrement (PN-Counter)
- registers: last-writer-wins and multi-value
- sets: grow-only, two-phase, and observed-remove (add wins)
- maps from any ordered key to any CRDT
- vector clocks
- sequences, for text and other ordered content
- ways to combine CRDTs: pairs, `Option` and lists

## Example

Two replicas count separately and then exchange states. The order of the exchange does not matter:

```
import "src/GCounter.taut"
import "src/Semilattice.taut"

def replica_a(): List(Nat) := GCounter::increment(GCounter::increment(Nil, Zero), Zero)
def replica_b(): List(Nat) := GCounter::increment(Nil, Succ(Zero))

theorem replicas_converge(): merge(replica_a(), replica_b()) = merge(replica_b(), replica_a()) {
  merge_comm(replica_a(), replica_b())
}

theorem replicas_value(): GCounter::value(merge(replica_a(), replica_b())) = 3 {
  Refl
}
```

More examples are in [`examples/`](examples):

- [`replicas.taut`](examples/replicas.taut): counters, a PN-Counter, an LWW register and vector clocks.
- [`sets.taut`](examples/sets.taut): add-wins OR-Set, G-Set, 2P-Set, a map of counters, and a multi-value register.
- [`sequence.taut`](examples/sequence.taut): two replicas insert at the same place at the same time and get the same text however they merge.

## What is available

| Module | Provides |
| --- | --- |
| `Semilattice` | the class every CRDT here is an instance of: `merge`, the order `below`, and the delivery theorems below |
| `Instances` | `Nat` (max), `Yn`, pairs, `Option` and `List`. Composite types get the laws from their parts |
| `Latest` | keeps the larger value of any totally ordered type |
| `GCounter` | grow-only counter: `increment`, `value`, `increment_delta` |
| `PNCounter` | counter that goes up and down: `increment`, `decrement`, `value` (an `Int`), with deltas |
| `Register` | last-writer-wins register: `LwwRegister::new(time, value)`, `LwwRegister::get`. On equal times the larger value wins, so all replicas agree |
| `KeyMap` | a map from ordered keys to CRDTs, merging values under the same key. `KeyMap::put` and `KeyMap::get` (from `KeyMapFind`) write and read one key |
| `Sets` | `gset_add` / `gset_contains`, `twopset_add` / `twopset_remove` / `twopset_contains` |
| `ORSet` | observed-remove set where a concurrent add beats a remove: `ORSet::add`, `ORSet::remove`, `ORSet::contains` |
| `MVRegister` | multi-value register: `mvreg_write`, `mvreg_has`. Concurrent writes are both kept |
| `VClock` | vector clocks. `VClock::order` says whether two clocks are equal, before, after or concurrent |
| `Position`, `Sequence` | ordered sequences: `Sequence::insert`, `Sequence::delete`, `Sequence::read`. `Position::between` always makes a position between two others |

## Guarantees

For every CRDT here, `merge` is commutative, associative and idempotent. These facts are proved, not tested, and they hold for every type built from the parts above:

- `deliver_perm`: a replica reaches the same state whatever order it receives the same set of states in (strong eventual consistency).
- `deliver_twice`: receiving a state twice is the same as receiving it once.
- `deliver_swap`, `deliver_append`: receiving two batches in either order gives the same result.
- `below` is a partial order, `merge` is its least upper bound, and `merge` is monotone.

Each type also has its own proofs, for example:

- a counter's value grows by exactly 1 per increment, and merging an increment's delta is the same as the increment;
- in a 2P-Set, an element that was removed is never a member again after any merge;
- in an OR-Set, merging a concurrent add and remove keeps the element (`ORSet::add_wins`);
- `VClock::order` agrees with `below`;
- a new sequence position can always be made between two others, and reads come out in position order after any operations and merges.

The theorems are in the same files as the code; start with `src/Semilattice.taut`.

## Checking

```sh
tauto check --tote .
for f in examples/*.taut; do tauto check "$f"; done
tauto lint src/*.taut examples/*.taut
tauto fmt --check src/*.taut examples/*.taut
```

## License

Licensed under either of [MIT](LICENSE-MIT) or [Apache-2.0](LICENSE-APACHE), at your option.
