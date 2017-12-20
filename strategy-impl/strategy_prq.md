Strategy PRQ
============

```{.go .lib}
package decision

import (
	"sync"
	"time"

	wantlist "github.com/ipfs/go-ipfs/exchange/bitswap/wantlist"
	pq "github.com/ipfs/go-ipfs/thirdparty/pq"

	cid "gx/ipfs/QmNp85zy9RLrQ5oQD4hPyS39ezrrXpcaa7R4Y9kxdWQLLQ/go-cid"
	peer "gx/ipfs/QmXYjuNuxVzXKJCfWasQk1RqkhVLDM9jtUKhqc2WPQmFSB/go-libp2p-peer"
)
```

Type and Constructor
--------------------

```{.go .lib}
// verify interface implementation
var _ peerRequestQueue = &strategy_prq{}

type strategy_prq struct {
	lock     sync.Mutex
	pQueue   pq.PQ
	taskMap  map[string]*peerRequestTask
	partners map[peer.ID]*activePartner
	rrq      *RRQueue

	frozen map[peer.ID]*activePartner
}

func newStrategyPRQ(strategy Strategy) *strategy_prq {
	return &strategy_prq{
		taskMap:  make(map[string]*peerRequestTask),
		partners: make(map[peer.ID]*activePartner),
		frozen:   make(map[peer.ID]*activePartner),
		pQueue:   pq.New(partnerCompare),
		rrq:      newRRQueue(strategy),
	}
}
```

`tl.initRound()` sets up the initial conditions for a single round-robin round.
For every peer with tasks in their task queue, a corresponding `RRPeer` is
created. Each `RRPeer` lasts for a single round-robin round. The `RRPeer` is
allocated resources based on the peer's current weight relative to the total of
all peers.

**TODO**: Track/test `totalWeight` to ensure it's updated correctly. As a part
of this, check whether `totalWeight` includes peers who don't currently have
tasks (it may not, because it's only updated when `prq.Push()` is called...not
certain).

```{.go .lib}
func (tl *strategy_prq) initRound() {
	tl.rrq.resetAllocations()
	i := 0
	for id, weight := range tl.rrq.weights {
		i += 1
		if tl.partners[id].taskQueue.Len() > 0 {
			tl.rrq.addPeer(id, weight)
		}
	}
}
```

Push
----

`tl.Push(entry, to, receipt)` updates peer `to`'s task queue with `entry`.

```{.go .lib}
// Push currently adds a new peerRequestTask to the end of the list
func (tl *strategy_prq) Push(entry *wantlist.Entry, to peer.ID, receipt *Receipt) {
	//to := l.Partner // get from receipt?
	tl.lock.Lock()
	defer tl.lock.Unlock()
	
	partner, ok := tl.partners[to]
	
	if !ok {
		partner = newActivePartner()
		tl.pQueue.Push(partner)
		tl.partners[to] = partner
	}

	partner.activelk.Lock()
	defer partner.activelk.Unlock()
```

If the partner already has the block, we do nothing.

```{.go .lib}
	if partner.activeBlocks.Has(entry.Cid) {
		return
	}
```

If the CID is already in the task queue, we update the task and return.

```{.go .lib}
	if task, ok := tl.taskMap[taskKey(to, entry.Cid)]; ok {
		task.Entry.Priority = entry.Priority
		partner.taskQueue.Update(task.index)
		return
	}
```

Otherwise, we add the task to the peer's taskQueue.

```{.go .lib}
	task := &peerRequestTask{
		Entry:   entry,
		Target:  to,
		created: time.Now(),
		Done: func() {
			tl.lock.Lock()
			partner.TaskDone(entry.Cid)
			tl.pQueue.Update(partner.Index())
			tl.lock.Unlock()
		},
	}

	partner.taskQueue.Push(task)
	tl.taskMap[task.Key()] = task
	partner.requests++
	tl.pQueue.Update(partner.Index())
```

Finally, we update the round-robin queue with the peer's current ledger
(`receipt`). When peers are allocated for the next round-robin round, the weight
set by the most recent call to `updateWeight` for each peer will be used in
allocating peers.

```{.go .lib}
	tl.rrq.updateWeight(to, receipt)
}
```

Pop
---

`tl.Pop()` returns the next `peerRequestTask` to be served. Much of the
round-robin logic is handled in the call to `tl.nextTask()`.

```{.go .lib}
// Pop 'pops' the next task to be performed. Returns nil if no task exists.
func (tl *strategy_prq) Pop() *peerRequestTask {
	tl.lock.Lock()
	defer tl.lock.Unlock()

	// get the next peer/task to serve
	rrp, task := tl.nextTask()
	if task == nil {
		return nil
	}
	partner := tl.partners[rrp.id]

	// start the task
	partner.StartTask(task.Entry.Cid)
	// update the relevant state
	partner.requests--
```

Update the peer's round-robin state. If the peer's allocation value hits zero,
then they have been served all of the data that they've been allocated for this
round so we remove the from the round-robin queue. Otherwise, if the amount of
data they've been consecutively served has reached the peer burst limit, then we
move the peer to the end of the round-robin queue.

**TODO**: What happens with the logic around `served` if there's only one peer
in the queue?

```{.go .lib}
	rrp.allocation -= task.Entry.Size
	rrp.served += task.Entry.Size

	if rrp.allocation == 0 {
		// peer has reached allocation limit for this round, remove peer from queue
		tl.rrq.pop()
	} else if rrp.served == tl.rrq.peerBurst {
		// peer has requests but burst limit reached, move peer to end of queue
		rrp.served = 0
		tl.rrq.shift()
	}
	return task
}
```

### nextTask

`tl.nextTask (rrp, task)` returns the next task to serve, along with the
corresponding peer -- unless there is no next task, in which case it returns
`(nil, nil)`.

```{.go .lib}
// nextTask() uses the `RRQueue` and peer `taskQueue`s to determine the next
// request to serve
func (tl *strategy_prq) nextTask() (rrp *RRPeer, task *peerRequestTask) {
	if tl.pQueue.Len() == 0 {
		return nil, nil
	}
```

We have to check whether there are any peers in the current round-robin queue.
If there aren't, then we start a new round-robin round. If there are still no
peers in the queue, then there are either 1. no requests to serve, or 2. no
peers with a round-robin weight greater than 0 (**TODO**: check this, and
determine whether that's okay), and thus we return nothing.

```{.go .lib}
	if tl.rrq.numPeers() == 0 {
		// may have finished last RR round, reallocate requests to peers
		tl.initRound()
		if tl.rrq.numPeers() == 0 {
			// if allocations still empty, there are no requests to serve
			return nil, nil
		}
	}
```

Once we know there are peers with tasks to be served in the current round-robin
queue, we search for the first valid peer + task in the queue. Each of the steps
is explained below, but here we give a summary. Peer `peer` and task `task` are
a valid pair if:

1.  **TODO**: write out details below then fill these steps in

```{.go .lib}
	// figure out which peer should be served next
	for tl.rrq.numPeers() > 0 {
		rrp = tl.rrq.head()

		// is this necessary? there shouldn't be nil RRPs
		//if rrp == nil {
		//tl.rrq.pop()
		//continue
		//}

		task := tl.nextValidTask(tl.partners[rrp.id])
		if task == nil {
			tl.rrq.pop()
			continue
		}

		if !tl.checkTask(rrp, task) {
			continue
		}

		delete(tl.taskMap, task.Key())
		return rrp, task
	}
	return nil, nil
}
```

### nextValidTask

`tl.nextValidTask(partner)` returns the first task in `partner`'s queue

1.  that is not marked as trash, and
2.  whose size doesn't equal or exceed the round-robin queue's peer burst limit
    -- if it did, then it would never be served as the peerBurst limits how much
    data a peer can be served consecutively.

If either of the above criteria are not met, then the task is deleted from the
task map.

**TODO**: need to consider how to handle the second criterion. if we didn't do
this check or allowed 1 larger-than-peerBurst task to be served, then it might
be game-able (if a peer just always sent absurdly large requests). might be best
to just log this and ensure peerBurst is large enough.

```{.go .lib}
// get first non-trash task
func (tl *strategy_prq) nextValidTask(partner *activePartner) *peerRequestTask {
	for partner.taskQueue.Len() > 0 {
		task := partner.taskQueue.Pop().(*peerRequestTask)
		// delete task if it's trash or too large to serve
		// TODO: log if task is too large?
		if task.trash || task.Entry.Size > tl.rrq.peerBurst {
			delete(tl.taskMap, task.Key())
			continue
		}
		return task
	}
	return nil
}
```

### checkTask

`tl.checkTask(rrp, task)` checks whether serving `task` would exceed `rrp`'s
current allocation or consecutive serve limit. If so, it returns `true`;
otherwise, it returns `false`.

```{.go .lib}
func (tl *strategy_prq) checkTask(rrp *RRPeer, task *peerRequestTask) bool {
	return tl.checkTaskVsAllocation(rrp, task) && tl.checkTaskVsPeerBurst(rrp, task)
}
```

### checkTaskVsAllocation

`tl.checkTaskVsAllocation(rrp, task)` checks whether serving `task` would exceed
`rrp`'s current round-robin allocation. If so, then it pushes the task back onto
the queue, removes the peer from the queue, and returns `false`. (If the peer
weren't removed from the queue, then they'd be stuck in this situation forever.
Note, though, that if their allocation does not sufficiently increase in any
future round and this task stays at the front of their queue, they'll forever
loop in this state.) Otherwise, it return `true`.

```{.go .lib}
func (tl *strategy_prq) checkTaskVsAllocation(rrp *RRPeer, task *peerRequestTask) bool {
	// check whether |task| exceeds peer's round-robin allocation
	if task.Entry.Size > rrp.allocation {
		tl.partners[rrp.id].taskQueue.Push(task)
		tl.rrq.pop()
		return false
	}
	return true
}
```

### checkTaskVsPeerBurst

`tl.checkTaskVsPeerBurst(rrp, task)` checks whether serving `task` exceeds
`rrp`'s consecutive serve limit. If so, then it pushes the task back onto the
queue, resets `rrp`'s consecutive serves to 0, moves the peer to the back of the
queue, and returns `false`. Othewrise, it returns `true`.

```{.go .lib}
func (tl *strategy_prq) checkTaskVsPeerBurst(rrp *RRPeer, task *peerRequestTask) bool {
	// check whether serving this block would exceed the consecutive serve limit
	if rrp.served+task.Entry.Size > tl.peerBurst {
		tl.partners[rrp.id].taskQueue.Push(task)
		rrp.served = 0
		tl.rrq.shift()
		return false
	}
	return true
}
```

Remove
------

`tl.Remove(k, p)` removes the task for CID `k` from `p`'s task queue. The
round-robin queue is unaffected by this function.

```{.go .lib}
// Remove removes a task from the queue
func (tl *strategy_prq) Remove(k *cid.Cid, p peer.ID) {
	tl.lock.Lock()
	t, ok := tl.taskMap[taskKey(p, k)]
	if ok {
		// remove the task "lazily"
		// simply mark it as trash, so it'll be dropped when popped off the
		// queue.
		t.trash = true

		// having canceled a block, we now account for that in the given partner
		partner := tl.partners[p]
		partner.requests--

		// we now also 'freeze' that partner. If they sent us a cancel for a
		// block we were about to send them, we should wait a short period of time
		// to make sure we receive any other in-flight cancels before sending
		// them a block they already potentially have
		if partner.freezeVal == 0 {
			tl.frozen[p] = partner
		}

		partner.freezeVal++
		tl.pQueue.Update(partner.index)
	}
	tl.lock.Unlock()
}

func (tl *strategy_prq) fullThaw() {
	tl.lock.Lock()
	defer tl.lock.Unlock()

	for id, partner := range tl.frozen {
		partner.freezeVal = 0
		delete(tl.frozen, id)
		tl.pQueue.Update(partner.index)
	}
}

func (tl *strategy_prq) thawRound() {
	tl.lock.Lock()
	defer tl.lock.Unlock()

	for id, partner := range tl.frozen {
		partner.freezeVal -= (partner.freezeVal + 1) / 2
		if partner.freezeVal <= 0 {
			delete(tl.frozen, id)
		}
		tl.pQueue.Update(partner.index)
	}
}
```

Strategy
--------

A Bitswap strategy function provides a weight to each peer based on their
current Bitswap ledger. This value is then used to provided weighted allocations
in the round-robin queue.

```{.go .lib}
// takes in a peer's ledger, returns RR weight for that peer
type StrategyFunc func(r *Receipt) float64

type Strategy struct {
	weightFunction StrategyFunc
	// determines how much to reduce weight by when freezing peer
	freezeWeight float64
}

// example Strategy
var ReciprocationStrategy = Strategy{
	weightFunction: ProbSend,
	freezeWeight:   0.8,
}

// simple weighting function based on peer's ledger Value
func ProbSend(r *Receipt) float64 {
	if r.Value <= 0 {
		return 0
	}
	return r.Value
}
```
