---
layout: post
title:  "Coin-sliding Font Puzzles"
date:   2019-11-06
---

At the 13th [Gathering for
Gardner](https://www.gathering4gardner.org), [Erik
Demaine](https://erikdemaine.org/) gave a [talk about coin-sliding
puzzles](https://www.youtube.com/watch?v=hBIGDBXWDP0), where you
rearrange coins to form the shapes of different letters.

He also announced a site where you can try your hand at [all 2,664
puzzles](https://coinsliding.erikdemaine.org/). That site also has a
couple papers which prove that all these puzzles are solvable in
polynomial time -- but the _best_ solutions were not known.

Well -- obviously -- the first thing that occurred to me was whether
it would be possible to find the optimal solution to all the puzzles!

## The basic search

Unlike, say, a packing puzzle where you can add pieces one at a time
until they're all used, sliding coins around has no clear notion of
progress -- you can just go in circles forever. So a depth-first
search doesn't work. Instead we need a breadth-first search, which is
going to have to remember every board configuration it visits to avoid
going in circles.

I started with the 5x7 puzzles. For these, the board is 35 squares,
and there are 12 coins. A natural representation for the board is just a
35-bit integer with 12 bits set, corresponding to the locations of the
coins. That means there are (35 choose 12) = 834,451,800 possible
positions of the coins (not all of which are actually reachable). 

These are actually pretty reasonable numbers. 2<sup>35</sup> bits is
only 4GB, so we can remember all the boards we have already visited in
a simple bit array -- it's very inefficient since almost all the bits
correspond to integers which don't have exactly 12 bits set, but it's
also very simple and convenient.

It's also feasible to just store every board we ever generate, as a
64-bit number -- there are only 800 million of them, which takes about
6.5GB. And for a similar price, we can remember, for every board, a
"parent pointer" to the board it was generated from.

With all of that, the search proceeds as follows, from any given
starting point:
- Generate all legal moves from the current board
- Look each one up in the big bit array of already-seen boards. If we
have seen it before, then we can already get to that board in the same
number or fewer steps, so no need to consider it again. 
- If this board is new:
  - Add it to a list of boards generated at this level (number of
  moves from the start)
  - Also save a pointer to the board it was generated from
  - Set its bit in the already-seen array
- Compare this board to all the 'target' boards -- the solutions we
are trying to reach
  - If it matches, hooray! Walk the list of parent pointers back to
  the start to retrace the shortest path from the starting board to
  this target.

This was remarkably successful. From a given starting point, it took
only about 13 minutes to reach all 36 of the targets.

## On to 5x9

Needless to say, the method above does not scale well. The 5x9 puzzles
have 45 squares on the board, and 13 coins. (45 choose 13) is about 73
billion -- nearly two orders of magnitude more possible boards, and
too many to store a 64-bit number for each (on any machine I have
access to, anyway). And of course 2<sup>45</sup> of anything is just
way too much. So I needed a whole new bag of tricks for both space and
speed.

#### Storing the boards: combinatorial rank

This is the coolest thing I learned from this project. I needed a way
to remember some information about every board I visited. With 73
billion possible boards, I could only afford about one byte each. A
board is a 45-bit number with exactly 13 bits set -- which means these
numbers are scattered all over a 45-bit space. A hash table is usually
a good way of storing a bunch of arbitrary data, but then you need to
remember the hash key, which by itself is far more than I have room
for.

Enter the [combinatorial number
system](https://en.wikipedia.org/wiki/Combinatorial_number_system). If
you have (N choose k) different combinations of _k_ elements from a
set of size _N_, this can map each one to a unique number between 0
and (N choose k)-1 -- also known as its combinatorial rank. Presto!
This is exactly what I needed -- a way to map each unique board into a
tightly packed list.

To compute the combinatorial rank, we iterate over the set bits, and
for each set bit compute (B choose I), where _B_ is the position of
the set bit (from LSB which is bit 0), and _I_ is the index in the
list of set bits (in the range 1 .. _k_); and then we add all these
up. Note that bit positions start at 0 but set-bit indices start at 1. (Because it makes it work, that's why.)

To quickly compute the rank, I use a three-step lookup table, breaking
the number (rather arbitrarily) into three pieces of 20, 16, and 13
bits. The low bits are a simple lookup, but the rank of the
higher-order bits depends on how many 1s there are in the lower
sections, so I actually have 21 tables for the middle 16 bits (for
whether the low 20 bits have anywhere from 0 to 20 bits set), and 37
tables for the upper 13 bits.

The x86 POPCNT and TZCNT instructions are hugely useful here. The
easiest way to get these with gcc is use the built-in functions
`__builtin_popcountl` and `__builtin_ctzl`, and then compile with
`-march=native` or something similar to ensure gcc uses the
instructions.

#### Finding moves

A legal move in the game slides a coin to an empty spot that is
adjacent to at least two other coins. To quickly find legal moves, I
used a lookup table with 2<sup>20</sup> entries: look up a 20-bit
vector representing four rows of five cells, and the table outputs a
vector of which cells have two or more neighbors. But since you need
the surrounding rows to correctly count neighbors, only the middle two
rows are guaranteed to be correct (and the top or bottom row, if it is
the top or bottom row of the whole board, and so has no neighbors on
one side). So I take the 45-bit vector which represents the whole
board and look it up in four overlapping 20-bit pieces, which gives me
rows 0/1/2, 3/4, 5/6, and 7/8 respectively, and just bitwise-OR them
all together.

To find the actual moves, I consider each coin (set bit) in the
board. I remove the coin from the board and then use the lookup
described above to find all the places that it would be legal to put
it -- this ensures that a coin can't count as one of its own
neighbors. Then I put the coin in every legal spot, except the spot it
came from, if that was an option.

#### All together now

The rank allows me to pack all the combinations into an address space
from 0 to (45 choose 13)-1. I allocate one byte per combination (which
is still pretty big, about 68GB), and use it to store the level
(number of moves from the starting point) at which the combination was
found. I don't have room for a separate list of the combinations found
at the last level, so the breadth-first search for level N just scans
the entire 68GB array; if an element equals N-1, then the combination
at that rank was generated at the previous level. I invert the rank
process to turn that into the actual combination, and then generate
all legal moves from there and insert them into the table with level N
(if they do not already exist).

When I add a new combination to the table, I check it against the list
of targets. If it matches a target, great! But I don't have room for
parent pointers, so I have to reverse-engineer the path I took to get
here. I compute all reverse moves from the target, and look them up in
the array until I find one which was generated at level N-1. That
gives me one valid step backward toward the starting point, and I
repeat N times to get the full solution.

There are probably some further ways to optimize both the speed and
space -- packing the data more tightly, or compressing it, or
something. But I stopped when it was good enough to get the job
done. For the 5x9 puzzles, finding the shortest path to all targets
from one starting point took about 30 hours. With a few machines, I
was able to solve all 1,332 puzzles in about a week.

## Conclusions

I'm not going to post all the solutions, to avoid ruining the fun of
solving Erik's puzzles by hand. But overall this was a really fun
project with a satisfying conclusion. The general approach doesn't
scale well, but fortunately the puzzles Erik had designed were just
the right size to require some clever tricks but still be in
reach. One more row of cells on the board, or even one more coin, and
it probably wouldn't have been possible with my computers.

The combinatorial rank is a wonderful trick, and I'm looking forward
to finding another use for it! 
