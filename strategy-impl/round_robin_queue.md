Round-Robin Queue
=================

```{.go .lib}
package decision

import (
    "math"

	peer "github.com/libp2p/go-libp2p-peer"
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

```{.go .lib}
type RRPeer struct {
	id         peer.ID
	allocation int
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
-   `allocations` is a list of round-robin peers (`RRPeer`s) which maintain the
    state of each peer throughout a round-robin round. This is a list of
    `RRPeer`'s rather than a map `peerID` to corresponding allocation because we
r   treat this as a queue of peers that we want to maintain the order of.

```{.go .lib}
// Round Robin Queue
type RRQueue struct {
	roundBurst  int
	strategy    Strategy
	weights     map[peer.ID]float64
	allocations []*RRPeer
}

func newRRQueue(s Strategy) *RRQueue {
	return &RRQueue{
		roundBurst:  1000,
		strategy:    s,
		weights:     make(map[peer.ID]float64),
		allocations: []*RRPeer{},
	}
}

// TODO: accept config object
func newRRQueueCustom(s Strategy, burst int) *RRQueue {
	return &RRQueue{
		roundBurst:  burst,
		strategy:    s,
		weights:     make(map[peer.ID]float64),
		allocations: []*RRPeer{},
	}
}
```

Peer Management
---------------

`rrq.InitRound()` sets the initial conditions for the next round-robin round.
The weights are used to calculate each peer's allocation for this round.

```{.go .lib}
func (rrq *RRQueue) InitRound() {
	totalWeight := float64(0)
	for _, weight := range rrq.weights {
		totalWeight += weight
	}
	
    for id, weight := range rrq.weights {
    	allocation := int((weight / totalWeight) * float64(rrq.roundBurst))
    	if allocation <= 0 {
    		continue
    	}
    	rrp := &RRPeer{
    		id:         id,
    		allocation: allocation,
    	}
    	rrq.allocations = append(rrq.allocations, rrp)
    }
}
```

`rrq.UpdateWeight(id, receipt)` updates `id`'s weight with the value calculated
by the `strategy` function, using `receipt` as the input.

```{.go .lib}
// update peer's weight using their current receipt
func (rrq *RRQueue) UpdateWeight(id peer.ID, r *Receipt) {
	rrq.weights[id] = rrq.strategy(r)
}
```

`rrq.Pop()` removes the peer at the front of the queue and does not return that
peer, while `rrq.Head()` returns the peer at the front of the queue (if the
queue is not empty) but does not remove the peer from the queue.

```{.go .lib}
func (rrq *RRQueue) Pop() {
    if len(rrq.allocations) != 0 {
    	rrq.allocations = rrq.allocations[1:]
	}
}

func (rrq *RRQueue) Head() *RRPeer {
	if len(rrq.allocations) == 0 {
		return nil
	}
	return rrq.allocations[0]
}
```

`rrq.Shift()` moves the peer at the front of the queue to the end.

```{.go .lib}
func (rrq *RRQueue) Shift() {
    rrq.allocations = append(rrq.allocations[1:], rrq.allocations[0])
}
```

`rrq.ResetAllocations()` resets the `allocations` to an empty list. This should
be used before initializing a new round.

```{.go .lib}
func (rrq *RRQueue) ResetAllocations() {
	rrq.allocations = []*RRPeer{}
}
```

Utility Functions
-----------------

`rrq.NumPeers()` returns the number of peers who have allocations in the
currently active round.

```{.go .lib}
func (rrq *RRQueue) NumPeers() int {
	return len(rrq.allocations)
}
```

Strategy
--------

A Bitswap strategy function provides a weight to each peer based on their
current Bitswap ledger. This value is then used to provided weighted allocations
in the round-robin queue.

```{.go .lib}
// takes in a peer's ledger, returns RR weight for that peer
type Strategy func(r *Receipt) float64

// simple weighting function based on peer's ledger Value
func Simple(r *Receipt) float64 {
	if r.Value <= 0 {
		return 0
	}
	return r.Value
}

func Exp(r *Receipt) float64 {
    return 100 / (1 + math.Exp(2 - r.Value))
}
```
