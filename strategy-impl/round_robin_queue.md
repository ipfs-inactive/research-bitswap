Round-Robin Queue
=================

```go
package decision

import (
	"math"

	peer "gx/ipfs/QmXYjuNuxVzXKJCfWasQk1RqkhVLDM9jtUKhqc2WPQmFSB/go-libp2p-peer"
)
```

Types and Constructors
----------------------

An `RRPeer` represents the round-robin allocation for a single peer in a given
round. It is made up of the following values:

-   `id` is the peer ID of the peer this `RRPeer` represents.
-   `allocation` represents the remaining resources that this peer should
    receive in the current round. This value is initialized as the total amount
    of resources the peer is allocated for the round, and the peer is served
    until this value reaches zero *or* it's too small -- e.g. if the peer has 50
    bytes of allocation remaining but the next block to serve this peer is 1
    kilobyte, then we remove the peer from the current round.
-   `served` is the amount of data that the peer has been served consecutively.
    This value is used to ensure that a peer is served `peerBurst` resources
    consecutively. When serving the next task for this peer would increased
    `served` beyond `peerBurst`, the peer is moved to the back of the `RRQueue`
    and their `served` value is set to 0.

```go
type RRPeer struct {
	id         peer.ID
	allocation int
	served     int
}
```

An `RRQueue` represents a round-robin queue and is made up of the following
values and data structures:

-   `peerBurst` is a parameter that controls the amount of data a peer is sent
    consecutively within a single round-robin round. Units: **TODO**
-   `round` is a parameter that controls the total amount of data that is
    allocated to peers in a single round-robin round. Units: **TODO**
-   `strategy` is the Bitswap strategy function used to weight peers at the
    start of a round-robin round.
-   `weights` contains the relative weights for each peer that are used in
    allocating resources to peers at the start of each round-robin round.
-   `totalWeight` is the sum of all of the weights in `weights` and is used to
    normalize peer weights to the total when allocating resources to peers. This
    value is stored to avoid unecessarily recalculating it (**TODO**: necessary
    to keep this?).
-   `allocations` is a list of round-robin peers (`RRPeer`s) which maintain the
    state of each peer throughout a round-robin round.

```go
// Round Robin Queue
type RRQueue struct {
	peerBurst   int
	roundBurst  int
	strategy    Strategy
	weights     map[peer.ID]float64
	totalWeight float64
	allocations []*RRPeer
}

func newRRQueue(s Strategy) *RRQueue {
	return &RRQueue{
		peerBurst:   30,
		roundBurst:  1000,
		strategy:    s,
		weights:     make(map[peer.ID]float64),
		totalWeight: 0,
		allocations: []*RRPeer{},
	}
}

// TODO: accept config object
func newRRQueueCustom(pb, rb int, s Strategy) *RRQueue {
	return &RRQueue{
		peerBurst:   pb,
		roundBurst:  rb,
		strategy:    s,
		weights:     make(map[peer.ID]float64),
		totalWeight: 0,
		allocations: []*RRPeer{},
	}
}
```

**TODO**: continue from here

```go
func (rrq *RRQueue) addPeer(id peer.ID, weight float64) {
	allocation := int((weight / rrq.totalWeight) * float64(rrq.roundBurst))
	if allocation <= 0 {
		return
	}
	rrp := &RRPeer{
		id:         id,
		allocation: allocation,
		served:     0,
	}
	rrq.allocations = append(rrq.allocations, rrp)
}

func (rrq *RRQueue) updateWeight(id peer.ID, r *Receipt) {
	// get old weight
	old_weight, ok := rrq.weights[id]
	if !ok {
		old_weight = 0
	}
	// update peer's weight based on strategy func
	rrq.weights[id] = rrq.strategy.weightFunction(r)
	// add delta to totalWeight
	rrq.totalWeight += rrq.weights[id] - old_weight
}

func (rrq *RRQueue) resetAllocations() {
	rrq.allocations = []*RRPeer{}
}

func (rrq *RRQueue) freeze(id peer.ID) {
	rrq.weights[id] *= rrq.strategy.freezeWeight
}

func (rrq *RRQueue) thaw(id peer.ID, freezeVal int) {
	rrq.weights[id] /=
		math.Pow(rrq.strategy.freezeWeight, float64(freezeVal+1)/2)
}

func (rrq *RRQueue) fullThaw(id peer.ID, freezeVal int) {
	// TODO: may want more reversible way of resetting weight...maybe store original weights?
	rrq.weights[id] /=
		math.Pow(rrq.strategy.freezeWeight, float64(freezeVal))
}

func (rrq *RRQueue) numPeers() int {
	return len(rrq.allocations)
}

func (rrq *RRQueue) set(id peer.ID, allocation int) {
	for _, peer := range rrq.allocations {
		if peer.id == id {
			peer.allocation = allocation
		}
	}
}

func (rrq *RRQueue) shift() {
	var peer *RRPeer
	peer, rrq.allocations = rrq.allocations[0], rrq.allocations[1:]
	rrq.allocations = append(rrq.allocations, peer)
}

func (rrq *RRQueue) pop() {
	// TODO: bounds check
	rrq.allocations = rrq.allocations[1:]
}

func (rrq *RRQueue) head() *RRPeer {
	if len(rrq.allocations) == 0 {
		return nil
	}
	return rrq.allocations[0]
}

func newRRPeer(p peer.ID) *RRPeer {
	return &RRPeer{
		id:     p,
		served: 0,
	}
}

func (rrp *RRPeer) exceedsAllocation(size int) bool {
	return size > rrp.allocation
}
```
