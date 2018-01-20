Strategy PRQ
============

```{.go .lib}
package decision

import (
	"sync"
	"time"

	wantlist "github.com/ipfs/go-ipfs/exchange/bitswap/wantlist"
	pq "github.com/ipfs/go-ipfs/thirdparty/pq"

	peer "gx/ipfs/QmWNY7dV54ZDYmTA1ykVdwNCqC11mpU4zSUp6XDpLTH9eG/go-libp2p-peer"
	cid "gx/ipfs/QmeSrf6pzut73u6zLQkRFQ3ygt3k6XFT2kjdYP8Tnkwwyg/go-cid"
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
}

func newStrategyPRQ(strategy Strategy) *strategy_prq {
	return &strategy_prq{
		taskMap:  make(map[string]*peerRequestTask),
		partners: make(map[peer.ID]*activePartner),
		pQueue:   pq.New(partnerCompare),
		rrq:      newRRQueue(strategy),
	}
}

func newStrategyPRQCustom(strategy Strategy, burst int) *strategy_prq {
	return &strategy_prq{
		taskMap:  make(map[string]*peerRequestTask),
		partners: make(map[peer.ID]*activePartner),
		pQueue:   pq.New(partnerCompare),
		rrq:      newRRQueueCustom(strategy, burst),
	}
}
```

Push
----

`tl.Push(entry, receipt)` updates the task queue corresponding to `receipt.Peer`
with `entry`.

```{.go .lib}
// Push adds a new peerRequestTask to the end of the list
func (tl *strategy_prq) Push(entry *wantlist.Entry, receipt *Receipt) {
	to := peer.ID(receipt.Peer)
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
set by the most recent call to `rrq.updateWeight()` for each peer will be used
in allocating peers.

```{.go .lib}
	tl.rrq.UpdateWeight(to, receipt)
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
	partner.requests--
```

Update the peer's round-robin state. If the peer's allocation value hits zero,
then they have been served all of the data that they've been allocated for this
round so we remove the from the round-robin queue.

```{.go .lib}
	rrp.allocation -= task.Entry.Size

	if rrp.allocation == 0 {
		// peer has reached allocation limit for this round, remove peer from queue
		tl.rrq.Pop()
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
	if tl.rrq.NumPeers() == 0 {
		// may have finished last RR round, reallocate requests to peers
		tl.rrq.InitRound()
		if tl.rrq.NumPeers() == 0 {
			// if allocations still empty, there are no requests to serve
			return nil, nil
		}
	}
```

Once we know there are peers with tasks to be served in the current round-robin
queue, we search for the first valid peer + task in the queue.

```{.go .lib}
	// figure out which peer should be served next
	for tl.rrq.NumPeers() > 0 {
		rrp = tl.rrq.Head()

		task := tl.partnerNextTask(tl.partners[rrp.id])
		// nil task means this peer has no valid tasks to be served at the moment
		if task == nil {
			tl.rrq.Pop()
			continue
		}
```

Check whether serving `task` would exceed `rrp`'s current round-robin
allocation. If so, push the task back onto the queue and remove the peer from
the current round. (If the peer weren't removed from the round, then they'd be
stuck in this situation forever. Note, though, that if their allocation does not
sufficiently increase in any future round and this task stays at the front of
their queue, they'll forever loop in this state.)

```{.go .lib}
    	// check whether |task| exceeds peer's round-robin allocation
    	if task.Entry.Size > rrp.allocation {
    		tl.partners[rrp.id].taskQueue.Push(task)
    		tl.rrq.Pop()
    		continue
		}

		return rrp, task
	}
	return nil, nil
}
```

### partnerNextTask

`tl.partnerNextTask(partner)` returns the first task in `partner`'s queue that
is not marked as trash. Each task that is marked as trash is removed from the
task map.

```{.go .lib}
// get first non-trash task
func (tl *strategy_prq) partnerNextTask(partner *activePartner) *peerRequestTask {
	for partner.taskQueue.Len() > 0 {
		task := partner.taskQueue.Pop().(*peerRequestTask)
		// delete task if it's trash
		if task.trash {
		    task = nil
			continue
		}
		return task
	}
	return nil
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
		//
		// TODO: figure out how to implement this for RRQ, e.g. move partner to end of
		// RRQ so that they still get their allocations but go at the end of the round
		// (but then also have to decide what to do if partner only peer in queue
		/*if partner.freezeVal == 0 {
			tl.frozen[p] = partner
		}*/

		//partner.freezeVal++
		tl.pQueue.Update(partner.index)
	}
	tl.lock.Unlock()
}
```

Helpers
-------

```{.go .lib}
func (tl *strategy_prq) allocationForPeer(id peer.ID) int {
    for _, rrp := range tl.rrq.allocations {
        if rrp.id == id {
            return rrp.allocation
        }
    }
    return 0
}
```
