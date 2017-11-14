---
title: Bitswap Analysis
subtitle: 3 Strategies
author: David Grisham
header-includes:
    - \usepackage{mathtools}
    - \DeclarePairedDelimiter\abs{\lvert}{\rvert}
    - \newcommand{\Nbhd}[1]{\mathcal{N}_{#1}}
...

In this document, we analyze 3 strategies for a simple 2-player Bitswap
infinitely repeated game.

**TODO**: finish this intro. Might want to mention the paragraph below, but have
to be clear that the first section is more general than the specific case done
in the analysis

In a given round, a player's utility is dependent on the actions each of the
players took in the previous round -- in other words, the debt ratio only
considers the immediately preceding round, rather than all previous rounds.

Actions and Utility Functions
=============================

A player has two possible actions: reciprocate ($R$) or defect ($D$). The
utility function for player $i$ at time $t$ $u_i^t$:

$$
u_i^t = \sum_{j \in \Nbhd{i}} \delta_{a_j^{t}R} S_j(d_{ji}, \mathbf{d}_j^{-i}) B
         - \delta_{a_i^t R} B \\
$$

where

-   $\Nbhd{i}$ is the neighborhood of user $i$ (i.e. the set of peers $i$ is
    connected to)
-   $a_i^t \in \{R, D\}$ is the action user $i$ takes in round $t$
-   $\delta{ij}$ is the kronecker delta function
-   $d_{ji}^t$ is the reputation of user $i$ as viewed by peer $j$ (also
    referred to as the *debt ratio* from $i$ to $j$) in round $t$
-   $S_j(d_{ij}, \mathbf{d}_j^{-i}) \in \{0, 1\}$ is the *strategy function* of
    user $j$. This function considers the relative reputation of peer $i$ to the
    rest of $j$'s peers, and returns a weight for peer $i$. This weight is used
    to determine what proportion of $j$'s bandwidth to give to peer $i$.
-   $B$ is the (constant) amount of bandwidth that a user has to offer in a
    given round. We make the simplifying assumption that the users are
    homogeneous in this value, so they all have the same amount of bandwidth to
    offer.

**TODO**: explain $u_i^t$ in plain english

The terms *strategy* and *strategy function* are defined as:

-   A *strategy* is meant in the standard game-theoretical sense, which is a
    predetermined set of actions that a user will take in a game (potentially
    dependent on that user's previous payoffs, the actions of its peers, etc.).
-   A *strategy function* is a term used to specify the weighting function that
    a user uses when running the Bitswap protocol to determine how much
    bandwidth it wants to allocate to each of its peers whenever it's playing
    the $R$ strategy.

We can write the debt ratio $d_{ij}$ in terms of the number of bits exchanged
between peers $i$ and $j$:

$$
d_{ji}^t = \frac{b_{ji}^{t-1}}{b_{ij}^{t-1} + 1}
$$

where $b_{ij}^{t-1}$ is the total number of bits sent from $i$ to $j$ from round
$0$ through round $t-1$ (so, all rounds prior to round $t$).

We can define $b_{ij}^{t+1}$ in terms of $b_{ij}^{t}$ and $\delta_{a_i^t R}$ as
follows:


$$
b_{ij}^{t+1} = b_{ij}^t + \delta_{a_i^t R} S_i(d_{ij}, \mathbf{d}_i^{-j}) B
$$

So, the total number of bits sent from $i$ to $j$ increases by
$S_i(d_{ij}, \mathbf{d}_i^{-j}) B$ (the proportion of $i$'s total bandwidth that
$i$ allocates to $j$) if and only if peer $i$ reciprocated in round $t$ (i.e.,
$a_i^t = R \implies \delta_{a_i^t R} = 1$).

We can use this to write out $d_{ij}^{t+1}$ in terms of values from round $t$.

$$
d_{ij}^{t+1} = \frac{b_{ij}^{t}
                  + \delta_{a_i^t R} S_i(d_{ij}, \mathbf{d}_i^{-j}) B}
               {b_{ji}^t + \delta_{a_j^t R} S_j(d_{ji}, \mathbf{d}_j^{-i}) B + 1}
$$

Analysis
========

The **strategy function** that user $j$ will use to weight some peer $i$ is:

$$
S_j(d_{ji}, \mathbf{d}_j^{-i}) = \frac{d_{ji}}
    {\sum_{k \in \Nbhd{j}} d_{jk}}
$$



$$
u_i^t = \sum_{j \in \Nbhd{i}} \frac{\delta_{a_j^{t}R} d_{ji}^t}
             {\sum_{k \in \Nbhd{j}} d_{jk}^t} B
         - \delta_{a_i^t R} B \\
$$

**TODO**: Mention that $d_{ij}^t$ only considers previous round (and ensure
explanation is consistent with analysis below)

We now consider 3 strategies: tit-for-tat, pavlov, and grim-trigger. For each of
these, we'll determine whether the strategy is an SPNE for the 2-player game.

Tit-for-Tat
-----------

**TODO**

Grim-Trigger
------------

**TODO**

Pavlov
------

**TODO**

Discussion
----------

**TODO**: compare results to that of prisoner's dilemma

**TODO**: discuss next steps (more peers)
