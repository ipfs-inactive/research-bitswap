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
infinitely repeated game. In a given round, a player's utility is dependent on
the actions each of the players took in the previous round -- in other words,
the debt ratio only considers the immediately preceding round, rather than all
previous rounds.

TODO: finish this intro

Actions and Utility Functions
=============================

A player has two possible actions: reciprocate (R) or defect (D). The utility
functions for player $i$ at times $t$ and $t+1$ are given by $u_i^t$ and
$u_i^{t+1}$, resp.

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
-   $\mathbf{d}_j^{-i}$ is the vector of the debt ratios $j$ has stored for each
    of its peers, not including peer $i$
-   $S_j(d_{ij}, \mathbf{d}_j^{-i}) \in \{0, 1\}$ is the *strategy function* of
    user $j$. This function considers the relative reputation of peer $i$ to the
    rest of $j$'s peers, and returns a weight for peer $i$. This weight is used
    to determine what proportion of $j$'s bandwidth to give to peer $i$.
-   $B$ is the (constant) amount of bandwidth that a user has to offer in a
    given round. We make the simplifying assumption that all users are
    homogeneous in this value, so they all have the same amount of bandwidth to
    offer.

We can write the debt ratio $d_{ij}$ in terms of the number of bits exchanged
between peers $i$ and $j$:

$$
d_{ij}^t = \frac{b_{ij}^{t-1}}{b_{ji}^{t-1} + 1}
$$

where $b_{ij}^{t-1}$ is the total number of bits sent from $i$ to $j$ from round
$0$ through round $t-1$ (so, all rounds prior to round $t$).

Analysis
========

We'll analysis 3 strategies under the following conditions:

TODO: define strategy function

TODO: single-round-lookbehind condition

TODO: write denominator of equation below

$$
u_i^t = \sum_{j \in \Nbhd{i}} \frac{\delta_{a_j^{t}R} d_{ji}^t}
             {\sum_{k \in \Nbhd{j}} d_{jk}^t} B
         - \delta_{a_i^t R} B \\
$$

TODO: show debt ratio for $t+1$

TODO: denominator

$$
u_i^{t+1} = \sum_{j \in \Nbhd{i}} \frac{\delta_{a_j^{t+1}R} \frac{b_{ij}^t +
            \delta_{a_i^{t}R} \frac{B}{\abs{\Nbhd{i}}}}{b_{ji}^t
                + \delta_{a_j^{t}R}\frac{B}{\abs{\Nbhd{j}}} + 1}}
            {1}
$$

We now consider 3 strategies: tit-for-tat, pavlov, and grim-trigger. For each of
these, we'll determine whether the strategy is an SPNE for the 2-player game.

Tit-for-Tat
-----------


Grim-Trigger
------------


Pavlov
------


TODO: compare results to that of prisoner's dilemma

TODO: discuss next steps (more peers)

