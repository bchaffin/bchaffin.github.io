---
layout: post
title:  "Matchstick Squares"
date:   2019-10-08
---

OEIS sequence [A294249](https://oeis.org/A294249) gives the number of
matchsticks needed to make all squares from size 1x1 up to NxN
simultaneously. The squares are allowed to overlap, so the key to good
solutions is to reuse matches in as many different squares as
possible.

I'm grateful to [Colin
Wright](https://www.solipsys.co.uk/new/ColinWright.html) for sending
this one my way.

The OEIS has a [full list](https://oeis.org/A294249/a294249.txt) of
what I believe to be the optimal answers up through 46 squares, and
some best-known ones up through 63 squares. There's also [more
information](matchstick-sols.html) about which solutions are found
with which of the searches described below.

Here are the first few best solutions:

    1─┐ -- A 1x1 square is simple, just 4 matches.
    └─┘

    2─┬─┐ 
    ├─1.│ -- A 2x2 square with the 1x1 in the corner. 10 matches.
    └───┘ 

    3───1─┐ 
    │. .├─┤ -- That's clever, the 1x1 tucks into the gap between the 2x2
    ├───2.│    and 3x3, using only one extra match. 17 matches total.
    └─────┘

    4─────┬─┐ 
    │. . .│.│ -- Here the 2x2 and 3x3 are offset so that the 1x1 is 
    │. .2─┼─┤    created without any additional matches at all. 26 total.
    ├───1─3.│ 
    └───┴───┘ 

## The search

I didn't find any especially clever ways of reducing the search space
-- I'm just doing a depth-first search placing squares. So most of my
effort went into trying to make that as fast as possible.

I did some early searches (up to about N=30) while allowing the
squares to roam free over a large area, to see if there were ever good
solutions where all squares are not contained within the largest
square. I didn't find any, as you might expect -- packing the matches
densely gives the best chances of reusing them for later squares, so
it's hard to see how sticking outside the boundary of the largest
square could be beneficial. The larger searches assumed that
everything is inside the largest square. I think that's very likely to
be true, but it remains unproven.

With that assumption, the area we're working in has N+1 rows and N+1
columns, each of which has N segments that can be occupied by
matches. I represent the rows as bit vectors, in an array of N+1
64-bit integers, and the same for the columns. (This uses more bits
than necessary since there is wasted space at the end of each 64-bit
value, but packing them together complicates the code so much that
it's actually slower.)

### Avoiding symmetry

Squares have lots of symmetry, and we can save a lot of work by making
sure our search doesn't find solutions which are simple reflections or
rotations of each other.

The largest and second-largest squares are just in a fixed position,
sharing a top left corner. (If we allow smaller squares to roam
outside the largest one, then the second square's corner is limited to
one octant.) A single square has 8-fold symmetry, but with just the
first two squares placed we have lost symmetry from left to right, top
to bottom, and across one diagonal. All we have left is mirror
symmetry across the main top left-to-bottom right diagonal:

    4─────┬─┐ 
    │. . .│.│
    │. . .│.│
    ├─────┘.│ 
    └───────┘ 

After that, the diagonal symmetry will be preserved as long as all
squares are placed with their corners along the main diagonal. If the
current state of the board is symmetrical, we'll only place a square
in locations on or above the main diagonal. Once there is any square
off the diagonal, we'll have to consider all placements for all
smaller squares. 

### Pre-compute everything

A square in a particular location can be represented as the `x,y`
coordinates of its top-left corner, and two bitmasks: one for the
spaces its top and bottom sides will occupy in the rows, and one for
the spaces its left and right sides will occupy in the columns.

At the beginning of the program I make a list of all possible
positions of every square. I also remember whether that position is
symmetric (on the diagonal), and whether it is below the diagonal (and
so should be skipped if the board is still symmetrical).

### The inner loop

With all that work out of the way, the core of the search is a pretty
small recursive function. The board is represented as an array of
2*(N+1) 64-bit vectors for the rows and columns, with 1=empty and
0=full. When we place a square, we need to figure out how many new
matches were added, since it may overlap other squares. We can do this
by ANDing the pre-computed masks with the affected rows and columns,
and counting how many bits are set in the result. To count the bits we
can use gcc's `__builtin_popcountl`, which on a modern Intel processor
produces one 3-cycle instruction.

In general we just have to try every position of every square: even
choices which look bad now because they add a lot of matches might
turn out to allow some clever reuse later. The one exception is if we
can place this square without using any matches at all -- it was
already created by the intersections of larger squares. There's never
a case where choosing that option is a bad idea.

So instead of recursing immediately, we first run through all the
placements and build a list of ones which are not too expensive (they
don't use more matches than the best known solution for this N). If we
find a free placement, then we just take that; otherwise we recurse on
each of the options in the list.

### Setting boundaries

Having a good upper bound on the number of matches really cuts down
the search space. To get a pretty good bound, we can do some quick,
imperfect searches using heuristics.

The first cut is to require all squares to be placed on the main
diagonal, down to some small size (typically 20) to allow for some
variation in the endgame. This is a strong constraint and makes for a
fast search, which produces very good (sometimes even optimal)
results.

The bound from the diagonal search can be fed into the next step,
which is a greedy algorithm that, for each size square, considers only
the placements which add the minimum number of matches (but tries all
placements tied for that minimum).

A possible third level is to require that placing a square of size N
add no more than 2N matches -- in other words it must share at least
two full sides with existing squares. In practice this search is slow
enough that it's not worth any incremental improvement in the bound,
and so I just moved straight from the greedy search to the full
search. (In fact, all known optimal answers meet this constraint.)

Unfortunately, the possibility that square placements might be
completely free -- they might completely overlap existing matchsticks
-- makes it impossible to prune the search before actually exceeding
the upper bound on the total number of matchsticks.

## Results

As you get to larger sizes, some really interesting structure emerges
in the optimal solutions. There are two recognizable types: one with
most squares on the diagonal, and one with a sort of "ladder"
structure with staggered pairs or triples of squares 2 or 3 spaces
apart (I'm sure the number of squares in each group and the spacing
will grow for larger N). The ladder-type ones offer lots of
opportunities for clever placements of the smaller squares and so they
are much less regular than the diagonal ones.

Here's an example of a diagonal type:

    n=16: best score 190
    16──────────────────────4───┬─┬─┐ 
    │. . . . . . . . . . . .│. .│.│.│ 
    │. . . . . . . . . . . .│. .│.│.│ 
    │. . .13────────────────┼───1─3─┤ 
    │. . .│. . . . . . . . .└───2─┼─┤ 
    │. . .│. . . . . . . . . . .│.│.│ 
    │. . .│. . .10──────────────┼─┼─┤ 
    │. . .│. . .│. . . . . . . .│.│.│ 
    │. . .│. . .│. . . . . . . .│.│.│ 
    │. . .│. . .│. . .7─────────┼─┼─┤ 
    │. . .│. . .│. . .│. . . . .│.│.│ 
    │. . .│. . .│. . .│. . . . .│.│.│ 
    │. . .│. . .│. . .│. . . . .│.│.│ 
    │. . .│. . .│. . .│. . . . .│.│.│ 
    ├─────11────8─────5─────────14│.│ 
    ├─────12────9─────6───────────15│ 
    └─────┴─────┴─────┴─────────────┘ 
    190 matchsticks, 4 solutions
    12 squares on main diagonal; first one off diagonal is 4

And here's an example of a ladder type:

    n=21: best score 282
    21──────17──────────────────────────────┬─┐ 
    │. . . .│. . . . . . . . . . . . . . . .│.│ 
    │. .19──┼───15──────────────────────────┼─┤ 
    │. .│. .│. .│. . . . . . . . . . . . . .│.│ 
    │. .│. .│. .│. . . . . . . . . . . . . .│.│ 
    │. .│. .│. .│. . . . . . . . . . . . . .│.│ 
    │. .│. .│. .│. . . . . . . . . . . . . .│.│ 
    │. .│. .│. .│. . . . . . . . . . . . . .│.│ 
    13──2───9───┼───────────┬─┐. . . . . . .│.│ 
    │. .│. .│. .│. . . . . .│.│. . . . . . .│.│ 
    │. .11──┼───7───────────10┤. . . . . . .│.│ 
    │. .│. .│. .│. . . . . .│.│. . . . . . .│.│ 
    │. .│. .│. .│. . . . . .│.│. . . . . . .│.│ 
    │. .│. .│. .│. . . . . .│.│. . . . . . .│.│ 
    │. .│. .│. .│. . . . . .│.│. . . . . . .│.│ 
    │. .│. .│. .│. . . . . .│.│. . . . . . .│.│ 
    │. .│. .├───14──────────8─4───────┬─────16│ 
    │. .│. .└───┴───────────┼─1───────3─────┼─┤ 
    │. .│. . . . . . . . . .│.│. . . .│. . .│.│ 
    │. .│. . . . . . . . . .│.│. . . .│. . .│.│ 
    ├───18──────────────────12┼───────┼─────20│ 
    └───┴───────────────────┴─┴───────5───────┘ 
    282 matchsticks, 2 solutions
    5 squares on main diagonal; first one off diagonal is 17

These two types are often very close in the number of matchsticks they
use, and which one is best for a given N is hard to predict. In a
number of cases they tie; for example N=40 has both diagonal and
ladder optimal solutions, and N=34 has a diagonal solution, some
ladder-type ones which are found with the greedy placement algorithm
and also some other ladder-type ones which are only found with a full
search.

For gory details of the number of solutions of different types for all
sizes up to N=63, see [this table](matchstick-sols.html).

Overall, I had fun optimizing the search, and this code is quite
fast at its job. But I didn't really gain any significant insight into
the problem. There is clearly structure to the solutions, but I don't
understand it well enough to be able to construct an optimal solution
without a heavy-duty search.
