---
layout: post
title:  "Longest Topswops decks for sizes 20 and 21"
date:   2026-04-14
---


## The Topswops game

[Topswops](https://en.wikipedia.org/wiki/Topswops) is a game invented by John Conway. The procedure is simple: start with a deck of _N_ cards, numbered 1 through _N_, shuffled. Look at the number of the card on top, then reverse the order of that many cards on the top of the deck. Continue until the top card is a 1. For example, say we start with the 4-card deck **3, 1, 4, 2**. Then the game goes as follows, finishing after 4 moves:
```
3 1 4 2
4 1 3 2
2 3 1 4
3 2 1 4
1 2 3 4
```

The number of steps in the game for a given deck is the _length_ of the deck. The question, of course, is what is the maximum length of a deck of size _N_, which I'll call _L(N)_. It's known that there is a [quadratic lower bound](https://doi.org/10.1016/j.tcs.2010.08.011), and an exponential upper bound of the (_N_+1)th Fibonacci number (proof in the Wikipedia article). [OEIS](https://oeis.org/) sequence [A000375](https://oeis.org/A000375) gives the maximum length of the game for various deck sizes. It was most recently [extended to sizes 18 and 19](https://arxiv.org/abs/2103.08346) in 2021. Topswops was also the subject of an [AZsPCs programming contest](http://azspcs.com/Contest/Cards) in 2011, which produced longest-known decks for prime sizes up to 97.

## A Topswops search algorithm

Knuth gives several algorithms for finding a longest Topswops deck in Volume 4, section 7.2.1.2, solution to exercise 108. The last one, which he refers to as the "better" algorithm, is a forward search which fills in the value of cards only when it becomes necessary. We start with a deck of blank cards; then we choose a number for the topmost card and make topswops moves until another blank card is on top. By trying each value not already used for blank cards, we can recursively search all permutations while simultaneously making the topswop moves, so that we don't have to repeat the same initial moves for similar decks. And Knuth has a clever trick for keeping track of where in the initial deck each card started, so that when we choose a value for the top card we know which original card we're choosing: we label the cards in the initial deck with the negative of their position, so the starting deck is -1, -2, -3, etc. Then a blank card is anything negative, and when we choose a value for it we also keep track of which card we assigned in a separate initial deck.

We can prune the search in a few ways, to avoid considering all _N_! decks. The first two pruning strategies are straightforward:

- Never choose 1 as the top card until all other values have been used. Choosing 1 ends the game, so if there is any other choice available then obviously it will result in a longer deck.
- A longest deck must be a _derangement_; that is, it must not have any cards in their "home" position, i.e. card _M_ in the Mth position. If it did, we could do a backward move by flipping the top _M_ cards (thus putting _M_ on top of the deck), giving a deck whose length is 1 greater than the current deck.

The third pruning strategy gets more involved. Say we are searching for the best 10-card deck. At some point, maybe card 10 ends up in the last position. Now it can never move again, and effectively we continue the game with a 9-card deck. But if we already know _L(9)_, the maximum length of a 9-card deck, then that is an upper bound for the number of moves remaining; and if that upper bound plus the number of moves so far is not more than the current best 10-card deck, then we can stop searching right away, without filling in any remaining blank cards.

Knuth says that once the largest _M_ cards are in order at the end of the deck, we can use _L(N-M)_ as a bound. Others have observed that we can do better: we can apply the same bound if the largest _M_ cards are at the end of the deck _**in any order**_, which is a significant improvement for larger decks.

### More pruning

But we can do even better than that. Consider the 10-card search again: with the logic above, we would say that once 10 is in the last position, there can be at most _L(9)_=30 moves remaining. But can we do even that well? The longest 9-card deck is unique: `6 1 5 9 7 2 8 3 4`. We can append a 10 to that and then work backwards to see what 10-card deck (or decks) could lead to it, producing this series of decks:
```
6 1 5 9 7 2 8 3 4 10
10 4 3 8 2 7 9 5 1 6
3 4 10 8 2 7 9 5 1 6
```
After 2 moves, no card is in its home position, so no more backward moves are possible. Thus there is a unique 10-card deck, with length 32, which produces the maximum-length 9-card deck. Having done this tiny bit of work, and once we know that _L(10)_ > 32, we can tighten our bound by 1: for a maximum-length 10-card deck, once 10 is in the last position there can be at most _L(9)_-1=29 moves remaining.
