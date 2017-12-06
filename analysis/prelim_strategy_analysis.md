---
title: Bitswap Analysis
subtitle: 3 Strategies
author: David Grisham
header-includes:
    - \usepackage{mathtools}
    - \DeclarePairedDelimiter\abs{\lvert}{\rvert}
    - \newcommand{\Network}{\ensuremath{\mathcal{N}}}
    - \newcommand{\Nbhd}[1]{\mathcal{N}_{#1}}
...

**TODO**: Ensure periods at the end of all bullet points/lists are consistent

**TODO**: Figure out cleaner (more maintainable) solution to math mode spacing

---

In this paper, we analyze 3 strategies for a simple 2-player Bitswap infinitely
repeated game. We start by defining the system in the most general case, then do
an analysis on a system subject to simplifying constraints.

Bitswap is the data exchange protocol for the InterPlanetary File System (IPFS).
Our model is meant to reflect this use case of Bitswap as the decision engine
implemented by each user in a peer-to-peer distributed file system. In this
distributed file system of many users, each user is connect to a set of peers
that they trade data with. Every peer has a reputation with every other peer --
in other words, for every peer a user has, that user maintains a summary of
their interactions with that peer. Then, when deciding how to allocate their
resources among their peers at a given time, the user uses these aggregate
reputations to provide weights to each of their peers. For example, consider a
network of 3 peers, labeled $1$, $2$, and $3$. If peer $2$ sends twice as much
data to peer $1$ as peer $3$ sends to peer $1$ from time $0$ to $t-1$ (and peer
$1$ sends the same amount of data to both $2$ and $3$ over that time), then peer
$1$ might allocate $\frac{2}{3}$ of its bandwidth to peer $2$ and $\frac{1}{3}$
to peer $3$ at time $t$.

**TODO**: necessary to explicitly mention strategies here? trying to stay
informal, but might still be a good idea

System
======

We have a network \Network of $\abs{\Network}$ users. The terms *users* and
*players* will be used somewhat interchangeably, depending on context; the term
*peers* is used similarly, but primarily refers to users who are connected (and
thus participate as players in the same Bitswap game). Each of the users has a
neighborhood of peers, which is the set of users they are connected to. Each
pair of peers plays an infinitely repeated Bitswap game. The resource that users
have to offer to their peers is bandwidth. We make the following simplifying
assumptions about user's bandwidth:

1.  All users have the same amount of bandwidth to offer.
2.  A single user has the same amount of bandwidth to offer at each time step.

In other words, bandwidth is constant both in peer-space and in time. **TODO**:
worth saying it this way, or is 'peer-space' confusing? 

We also make the assumption that *all users always have unique data that all of
their peers want*. So, whenever a peer plays $R$, they'll always have some data
to send to all of their peers. This, of course, contrasts a more realistic
scenario where a peer chooses to reciprocate but simply does not have anything
that their peers want at the current time.

Actions and Utility Functions
-----------------------------

**TODO**: ensure lower bound for $t$ is consistently 0 (and not typo'd as 1)

A player has two possible actions: reciprocate ($R$) or defect ($D$). The
utility function for player $i$ at time $t$ $u_i^t$:

$$
u_i^t = \sum_{j \in \Nbhd{i}} \delta_{a_j^{t}R}\ 
              S_j(d_{ji}^t\,,\,\mathbf{d}_j^{-i,t})\ B
         \:-\:\delta_{a_i^t R}\ B \\
$$

where

-   $\Nbhd{i} \subseteq \Network$ is the neighborhood of user $i$ (i.e. the set
    of peers $i$ is connected to)
-   $a_i^t \in \{R, D\}$ is the action user $i$ takes in round $t$
-   $\delta{ij}$ is the kronecker delta function
-   $d_{ji}^t$ is the reputation of user $i$ as viewed by peer $j$ (also
    referred to as the *debt ratio* from $i$ to $j$) in round $t$
-   $\mathbf{d}_j^{-i,t} = (d_{jk}^t \mid \forall k \in \Nbhd{j}, k \neq i)$ is
    the vector of debt ratios for all of user $j$'s peers (as viewed by peer
    $j$) in round $t$, *excluding* peer $i$
-   $S_j(d_{ij}^t, \mathbf{d}_j^{-i,t}) \in \{0, 1\}$ is the *strategy function*
    of user $j$. This function considers the relative reputation of peer $i$ to
    the rest of $j$'s peers, and returns a weight for peer $i$. This weight is
    used to determine what proportion of $j$'s bandwidth to allocate to peer $i$
    in round $t$.
-   $B > 0$ is the (constant) amount of bandwidth that a user has to offer in a
    given round

The terms *strategy* and *strategy function* are defined as:

-   A *strategy* is meant in the standard game-theoretical sense, which is a
    predetermined set of actions that a user will take in a game (potentially
    dependent on that user's previous payoffs, the actions of its peers, etc.).
-   A *strategy function* is a term used to specify the weighting function that
    a user uses when running the Bitswap protocol to determine how much
    bandwidth it wants to allocate to each of its peers whenever it's playing
    the $R$ strategy.

Putting this all together, we see that the utility of peer $i$ in round $t$ is
the total amount of bandwidth that $i$ is allocated by its neighboring peers,
minus the amount of bandwidth that $i$ provides to its peers. If $i$
reciprocates, then we say that they provide a total of $B$ bandwidth to the
network; otherwise (when $i$ defects), $i$ provides $0$ bandwidth in that round.

We can write the debt ratio $d_{ij}$ in terms of the number of bits exchanged
between peers $i$ and $j$:

$$
d_{ji}^t = \frac{b_{ji}^{t-1}}{b_{ij}^{t-1}\:+\:1}
$$

where $b_{ij}^{t-1}$ is the total number of bits sent from $i$ to $j$ from round
$0$ through round $t-1$ (so, all rounds prior to round $t$).

We can define $b_{ij}^t$ in terms of $b_{ij}^{t}$ and $\delta_{a_i^t R}$ as
follows:

$$
b_{ij}^t = b_{ij}^{t-1}\:+\:
           \delta_{a_i^{t-1} R}\ S_i(d_{ij}^t\,,\,\mathbf{d}_i^{-j,t})\ B
$$

So, the total number of bits sent from $i$ to $j$ increases by
$S_i(d_{ij}^t, \mathbf{d}_i^{-j,t}) B$ (the proportion of $i$'s total bandwidth
that $i$ allocates to $j$) if and only if peer $i$ reciprocated in round $t-1$
(i.e., $a_i^{t-1} = R \implies \delta_{a_i^{t-1} R} = 1$).

Now we can write $d_{ij}^{t+1}$ in terms of values from round $t$.

$$
d_{ij}^{t+1} = \frac{b_{ij}^{t}\:+\:
                  \delta_{a_i^t R}\ S_i(d_{ij}^t\,,\,\mathbf{d}_i^{-j,t})\ B}
               {b_{ji}^t\:+\:\delta_{a_j^t R}\ S_j(d_{ji}^t\,,\,\mathbf{d}_j^{-i,t})
                 \ B\:+\:1}
$$

Analysis
========

For the purposes of this analysis, we make an additional assumption: each user's
neighborhood is constant -- so any given pair of peers is connected for the
entire repeated game. This means that the network topology is static as well.

We now consider consider a specific strategy function that user $j$ uses to
weight some peer $i$:

$$
S_j(d_{ji}^t\,,\,\mathbf{d}_j^{-i,t}) = \frac{d_{ji}^t}
    {\sum_{k \in \Nbhd{j}} d_{jk}^t}
$$



$$
u_i^t = \sum_{j \in \Nbhd{i}} \frac{\delta_{a_j^{t}R}\ d_{ji}^t}
           {\sum_{k \in \Nbhd{j}} d_{jk}^t}\ B
        \:-\:\delta_{a_i^t R}\ B \\
$$

As this is an infinitely repeated game, we want to be able to calculate the
discounted average payoff of player $i$, $U_i$.

$$
U_i = (1 - \epsilon_i) \sum_{t=1}^{\infty} \epsilon_i^{t-1}\ u_i^t(\mathbf{a}^t)
$$

where

-   $\epsilon_i \in (0, 1)$ is the *discount factor*, which characterizes how
    much player $i$ cares about their payoffs in future rounds relative to the
    current round.
-   $\mathbf{a}^t = (a_i^t \mid \forall i \in (1, \abs{\Network}))$ is the
    vector of each player's actions in round $t$.

Further, rather than $d_{ij}^t$ being an aggregate value over all rounds in
$[0, t)$, it will only take the immediately preceding round into account. In
other words:

$$
b_{ij}^t = \delta_{a_i^t R}\ S_i(d_{ij}^t\,,\,\mathbf{d}_i^{-j,t})\ B
$$

We now consider 3 strategies: tit-for-tat, pavlov, and grim-trigger. For each of
these, we'll determine whether the strategy is an subgame-perfect Nash
equilibrium (SPNE) for the 2-player infinitely repeated game. While it suffices
to show a single case of initial conditions that prove a strategy is not an
SPNE, all cases will be shown as they may prove useful in understanding the more
complex scenarios. (**TODO**: get others' opinions on whether wording in first
half of previous sentence is clear) (**TODO**: be sure to follow up on second
half of previous sentence later in analysis)

Note that since each user only has a single peer (the other user), when a player
plays $R$ they allocate all of their bandwidth to the other user. (**TODO**:
find best place for this statement)

To determine whether a given strategy is an SPNE for the 2-player infinitely
repeated (simplified) Bitswap game, we will do the following:

-   Consider an initial pair of actions at $t=0$, $(a_1^0, a_2^0)$.
-   Assume player 1 plays the strategy for all rounds.
-   Consider two cases:
    1.  Player 2 never deviates from the strategy.
    2.  Player 2 deviates from the strategy for a single round, at $t=1$, then
        plays the strategy in all future rounds.
-   Calculate player 2's payoff for the infinitely-repeated game in both cases.
    If player 2's payoff in case 2 ($U_2'$) is less than or equal to their
    payoff in case 1 ($U_2$) for all initial pairs of actions, then the strategy
    is an SPNE. Mathematically, a strategy is an SPNE if and only if
    $U_2' \leq U_2$ for all possible initial actions.

Tit-for-Tat
-----------

We start by analyzing the well-studied tit-for-tat (TFT) strategy. A player that
uses this strategy always takes the strategy that their peer took in the
previous round. So, if player 1 plays action $R$ ($D$) in round $t$, then player
2 will play action $R$ in round $t+1$ ($D$), and vice-versa.

Let's first consider case where the initial pair of actions is $(D, D)$.

### Case 1: $(D, D)$

Here, both players start by playing the $D$ strategy. We'll first consider the
case where player 2 does not deviate from TFT. The strategies at each round then
follow:

  $t$    $0$ $1$ $2$ $3$ $4$ ...
-------  --- --- --- --- --- ---
$a_1^t$   D   D   D   D   D  ...
$a_2^t$   D   D   D   D   D  ...

Since neither player deviates from TFT, they both continually play their
opponent's previous strategy -- the initial state is $(D, D)$, so each player
repeatedly plays $D$ in this instance.

We can calculate the payoff of player 2 in this case -- notice that, since
neither player is ever giving or receiving, they payoff at each round is $0$.

$$
u_2^t = 0\ \forall\ t\ \implies\ U_2 = 0
$$

Now we consider this case where player 2 deviates from TFT for 1 round, at
$t=1$. The resulting action sequence is then:

  $t$    $0$ $1$ $2$ $3$ $4$ ...
-------  --- --- --- --- --- ---
$a_1^t$   D   D   R   D   R  ...
$a_2^t$   D   R   D   R   D  ...

In this case, player 2 deviates from TFT for a single round and plays $R$ at
$t=1$ rather than $D$, then goes back to playing TFT for all rounds after that.
Player 1, on the other hand, never deviates from TFT.

We can calculate player 2's discounted average payoff in this second case,
$U_2'$. Note that we consider payoffs starting at $t=1$, not $t=0$ -- this is
because we're calculating the payoff that player 2 would perceive if they
deviated from TFT for a single round, and that decision is happening at $t=1$ in
our case. (**TODO**: work on the explanation in this last sentence)

At $t=1$, player 1 defects and player 2 reciprocates -- since there are only two
players (**TODO**: any other conditions, e.g. lookbehind length?), player 2
provides all of its bandwidth to player 1 when it plays $R$. Thus, player 1's
payoff is $u_1^1 = B$ and player 2's payoff at $t=1$ is $u_2^1 = -B$.

At $t=2$, the players swap strategies (because they're back to playing TFT). So
$u_1^2 = -B$ and $u_2^2 = B$.

The game alternates between these two states forever. Thus, the payoff for
player 2 in this case is:

\begin{align*}
U_2' &= -B + \epsilon_2 B - \epsilon_2^2 B + \epsilon_2^3 B - \ldots \\
     &= B (-(1 + \epsilon_2^2 + \epsilon_2^4 + \ldots)
           + \epsilon (1 + \epsilon_2^2 + \epsilon_2^4 + \ldots)) \\
     &= B (\epsilon - 1) \frac{1}{1- \epsilon_2^2} \\
     &= -\frac{B}{1+\epsilon_2}
\end{align*}

Given $U_2$ and $U_2'$, we can start to discern whether TFT might be an SPNE for
this game. We see that $U_2' < U_2$ -- this means that TFT *might* be an SPNE,
but we have to verify that this is the case for all other initial conditions as
well.

For the rest of the cases, we simply show the action sequences and the
discounted average payoff results

### Case 2: $(D, R)$

When player 2 doesn't deviate from TFT:

  $t$    $0$ $1$ $2$ $3$ $4$ ...
-------  --- --- --- --- --- ---
$a_1^t$   D   R   D   R   D  ...
$a_2^t$   R   D   R   D   R  ...

Thus,

\begin{align*}
U_2 &= B - \epsilon_2 B + \epsilon_2^2 B - \ldots \\
    &= \frac{B}{1 + \epsilon_2}
\end{align*}

When player 2 does deviate from TFT:

  $t$    $0$ $1$ $2$ $3$ $4$ ...
-------  --- --- --- --- --- ---
$a_1^t$   D   R   R   R   R  ...
$a_2^t$   R   R   R   R   R  ...

Both players give and receive $B$ bandwidth in all rounds $t \in [0, \infty)$,
so:

\begin{align*}
U_2' &= 0 - \epsilon_2 0 + \epsilon_2^2 0 - \ldots \\
     &= 0
\end{align*}

Thus, $U_2' < U_2$ in this case.

### Case 3: $(R, D)$

When player 2 doesn't deviate from TFT:

  $t$    $0$ $1$ $2$ $3$ $4$ ...
-------  --- --- --- --- --- ---
$a_1^t$   R   D   R   D   R  ...
$a_2^t$   D   R   D   R   D  ...

Thus,

\begin{align*}
U_2 &= -B + \epsilon_2 B - \epsilon_2^2 B + \ldots \\
    &= -\frac{B}{1 + \epsilon_2}
\end{align*}

When player 2 does deviate from TFT:

  $t$    $0$ $1$ $2$ $3$ $4$ ...
-------  --- --- --- --- --- ---
$a_1^t$   R   D   D   D   D  ...
$a_2^t$   D   D   D   D   D  ...

\begin{align*}
U_2' &= 0 - \epsilon_2 0 + \epsilon_2^2 0 - \ldots \\
     &= 0
\end{align*}

We get a differently result in this case, namely $U_2' > U_2$. Therefore, **TFT
is not an SPNE for this game**.

### Case 4: $(R, R)$

We now already proven that TFT is not an SPNE. This case gives that result as
well.

When player 2 doesn't deviate from TFT:

  $t$    $0$ $1$ $2$ $3$ $4$ ...
-------  --- --- --- --- --- ---
$a_1^t$   R   R   R   R   R  ...
$a_2^t$   R   R   R   R   R  ...

Thus,

\begin{align*}
U_2 = 0
\end{align*}

When player 2 does deviate from TFT:

  $t$    $0$ $1$ $2$ $3$ $4$ ...
-------  --- --- --- --- --- ---
$a_1^t$   R   R   D   R   D  ...
$a_2^t$   R   D   R   D   R  ...

Thus,

\begin{align*}
U_2' = \frac{B}{1 + \epsilon_2}
\end{align*}

We again get the result $U_2' > U_2$, which also indicates that TFT is not an
SPNE.

Grim-Trigger
------------

Now we consider the well-studied grim trigger (GT) strategy. A player that uses
this strategy plays $D$ in all rounds where their peer has previously played
$D$; otherwise, the player plays $R$. Formally, for the 2-player game, this
strategy is characterized by

$$
a_i^t =
  \begin{cases}
    D & \text{if } D \in (a_j^0, \ldots, a_j^{t-1}) \\
    R & \text{otherwise} 
  \end{cases}
$$

where $i, j \in \{1, 2\}$ and $i \neq j$.

### Case 1: $(D, D)$

In this first case, both players play $D$ in round $t=0$. As both users are
playing GT, each user defects for all succeeding rounds since their peer played
$D$ at some point in the past. Thus, the resulting strategy sequence is:

  $t$    $0$ $1$ $2$ $3$ $4$ ...
-------  --- --- --- --- --- ---
$a_1^t$   D   D   D   D   D  ...
$a_2^t$   D   D   D   D   D  ...

Thus,

$$
U_2 = 0
$$

When player 2 deviates from GT:

  $t$    $0$ $1$ $2$ $3$ $4$ ...
-------  --- --- --- --- --- ---
$a_1^t$   D   D   D   D   D  ...
$a_2^t$   D   R   D   D   D  ...

In this case, player 1's strategy is still $D$ for all rounds, since 2 played
$D$ at $t=0$. So player 2 ends up giving up $B$ bandwidth at $t=1$ and never
getting anything back, giving:

$$
U_2' = -B
$$

Thus, $U_2' > U_2$ in this case. Therefore, **GT is not an SPNE for this game**.

### Case 2: $(D, R)$

When player 2 does not deviate from GT:

  $t$    $0$ $1$ $2$ $3$ $4$ ...
-------  --- --- --- --- --- ---
$a_1^t$   D   R   D   D   D  ...
$a_2^t$   R   D   D   D   D  ...

Thus,

$$
U_2 = B
$$

When player 2 deviates from GT:

  $t$    $0$ $1$ $2$ $3$ $4$ ...
-------  --- --- --- --- --- ---
$a_1^t$   D   R   R   D   D  ...
$a_2^t$   R   R   D   D   D  ...

Thus,

\begin{align*}
U_2' &= B - B + \epsilon_2 B \\
     &= \epsilon_2 B
\end{align*}

In this case, $U_2' < U_2$.

### Case 3: $(R, D)$

When player 2 does not deviate from GT:

  $t$    $0$ $1$ $2$ $3$ $4$ ...
-------  --- --- --- --- --- ---
$a_1^t$   R   D   D   D   D  ...
$a_2^t$   D   R   D   D   D  ...

Thus,

$$
U_2 = -B
$$

When player 2 deviates from GT:

  $t$    $0$ $1$ $2$ $3$ $4$ ...
-------  --- --- --- --- --- ---
$a_1^t$   R   D   D   D   D  ...
$a_2^t$   D   D   D   D   D  ...

Thus,

$$
U_2' = 0
$$

In this case, $U_2' < U_2$.

### Case 4: $(R, R)$

When player 2 does not deviate from GT:

  $t$    $0$ $1$ $2$ $3$ $4$ ...
-------  --- --- --- --- --- ---
$a_1^t$   R   R   R   R   R  ...
$a_2^t$   R   R   R   R   R  ...

Thus,

$$
U_2 = 0
$$

When player 2 deviates from GT:

  $t$    $0$ $1$ $2$ $3$ $4$ ...
-------  --- --- --- --- --- ---
$a_1^t$   R   R   D   D   D  ...
$a_2^t$   R   D   R   D   D  ...

Thus,

\begin{align*}
U_2' &= B - \epsilon_2 B \\
     &= (1 - \epsilon_2) B
\end{align*}

In this case, $U_2' > U_2$, which would be proof that GT is not an SPNE for this
game had we not already proven that fact.

Pavlov
------

### Case 1: $(D, D)$


### Case 2: $(D, R)$


### Case 3: $(R, D)$


### Case 4: $(R, R)$


Discussion
----------

**TODO**: compare results to that of prisoner's dilemma

**TODO**: discuss next steps (more peers)
