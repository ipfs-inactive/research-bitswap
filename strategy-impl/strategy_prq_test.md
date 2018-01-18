Strategy PRQ Tests
==================

```{.go .lib}
package decision

import (
    "math"
	"math/rand"
	"sort"
	"strings"
    "testing"

	"github.com/ipfs/go-ipfs/exchange/bitswap/wantlist"
	"gx/ipfs/QmeDA8gNhvRTsbrjEieay5wezupJDiky8xvCzDABbsGzmp/go-testutil"
	u "gx/ipfs/QmPsAfmDBnZN3kZGSuNwvCNDZiHneERSKmRcFyG3UkvcT3/go-ipfs-util"
	//peer "gx/ipfs/QmWNY7dV54ZDYmTA1ykVdwNCqC11mpU4zSUp6XDpLTH9eG/go-libp2p-peer"
	cid "gx/ipfs/QmeSrf6pzut73u6zLQkRFQ3ygt3k6XFT2kjdYP8Tnkwwyg/go-cid"
)
```

Push, Pop
---------

`TestStrategyPushPop()` tests the `Push()`, `Pop()`, and `Remove()` functions
for the `strategy_prq`. This test was taken from the original `peerRequestQueue`
tests and is not specific to the round-robin functionality.

```{.go .lib}
func TestStrategyPushPop(t *testing.T) {
	prq := newStrategyPRQ(Simple)
	partner := testutil.RandPeerIDFatal(t)
	alphabet := strings.Split("abcdefghijklmnopqrstuvwxyz", "")
	vowels := strings.Split("aeiou", "")
	consonants := func() []string {
		var out []string
		for _, letter := range alphabet {
			skip := false
			for _, vowel := range vowels {
				if letter == vowel {
					skip = true
				}
			}
			if !skip {
				out = append(out, letter)
			}
		}
		return out
	}()
	sort.Strings(alphabet)
	sort.Strings(vowels)
	sort.Strings(consonants)

	// add a bunch of blocks. cancel some. drain the queue. the queue should only have the kept entries

    l := newLedger(partner).Receipt()
    l.Value = 1
	for _, index := range rand.Perm(len(alphabet)) { // add blocks for all letters
		letter := alphabet[index]
		t.Log(partner.String())

		c := cid.NewCidV0(u.Hash([]byte(letter)))
		prq.Push(&wantlist.Entry{Cid: c, Priority: math.MaxInt32 - index}, l)
	}
	for _, consonant := range consonants {
		c := cid.NewCidV0(u.Hash([]byte(consonant)))
		prq.Remove(c, partner)
	}

	var out []string
	for {
		received := prq.Pop()
		if received == nil {
			break
		}
		out = append(out, received.Entry.Cid.String())
	}

	// Entries popped should already be in correct order
	for i, expected := range vowels {
		exp := cid.NewCidV0(u.Hash([]byte(expected))).String()
		if out[i] != exp {
			t.Fatal("received", out[i], "expected", exp)
		}
	}
}
```
