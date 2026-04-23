---
layout: post
title:  "Longest Topswops decks for sizes 20 and 21"
date:   2026-04-14
---


## The Topswops game

[Topswops](https://en.wikipedia.org/wiki/Topswops) is a game invented by John Conway. The procedure is simple: start with a deck of _N_ cards, numbered 1 through _N_, shuffled. Look at the number of the card on top, then reverse the order of that many cards on the top of the deck. Continue until the top card is a 1. For example, say we start with the 5-card deck **3, 1, 4, 5, 2**. Then the game goes as follows, finishing after 7 moves:

- move 0: 3 1 4 5 2
- move 1: <span style="color:red">4 1 3</span> 5 2
- move 2: <span style="color:red">5 3 1 4</span> 2
- move 3: <span style="color:red">2 4 1 3 5</span>
- move 4: <span style="color:red">4 2</span> 1 3 5
- move 5: <span style="color:red">3 1 2 4</span> 5
- move 6: <span style="color:red">2 1 3</span> 4 5
- move 7: <span style="color:red">1 2</span> 3 4 5

The number of steps in the game for a given deck is the _length_ of the deck. The question, of course, is what is the maximum length of a deck of size _N_, which I'll call _L(N)_. It's known that there is a [quadratic lower bound](https://doi.org/10.1016/j.tcs.2010.08.011), and an exponential upper bound of the (_N+1_)th Fibonacci number (proof in the Wikipedia article). OEIS sequence [A000375](https://oeis.org/A000375) gives the maximum length of the game for various deck sizes. It was most recently [extended to sizes 18 and 19](https://arxiv.org/abs/2103.08346) in 2021. Topswops was also the subject of an [AZsPCs programming contest](http://azspcs.com/Contest/Cards) in 2011, which produced longest-known decks for prime sizes up to 97.

In the discussion below, I'll use "an _N-L deck_" to mean an _N_-card deck that takes _L_ moves; the example above is a 5-7 deck.

## A Topswops search algorithm

Knuth gives several algorithms for finding a longest Topswops deck in Volume 4, section 7.2.1.2, solution to exercise 108. The last one, which he refers to as the "better" algorithm, is a forward search which fills in the value of cards only when it becomes necessary. We start with a deck of blank cards; then we choose a number for the topmost card and make topswops moves until another blank card is on top. By trying each value not already used for blank cards, we can recursively search all permutations while simultaneously making the topswop moves, so that we don't have to repeat the same initial moves for similar decks. And Knuth has a clever trick for keeping track of where in the initial deck each card started: we label the cards in the initial deck with the negative of their position, so the starting deck is -1, -2, -3, etc. Then a blank card is anything negative, and when we choose a value for it we also separately keep track of which card we assigned in the initial deck.

### Pruning the search

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
After 2 moves, no card is in its home position, so no more backward moves are possible. Thus there is a unique 10-32 deck which produces the maximum-length 9-card deck. (Note that, in general, there could be multiple possible backward moves at one time, so we may have to do a real recursive search to find the longest backward chain of moves.) Having done this tiny bit of work, and once we know that _L(10)_ > 32, we can tighten our bound by 1: for a maximum-length 10-card deck, once 10 is in the last position the remaining deck cannot achieve _L(9)_, and there can be at most _L(9)_-1=29 moves remaining.

We can extend this method: during the size-9 search, we can do some extra work and look for 9-29 decks (one shorter than _L(9)_), and do the same backward search on those; if they also cannot achieve a new 10-card best, then we can reduce our bound by 1 again. But there's a pitfall here: our search wants to take advantage of the second pruning technique above and only look for derangements. This works when looking for maximum-length decks, but not when we also want to look for shorter decks. For example, there is exactly one 9-29 deck that our search will not find: the result of starting with the optimal 9-30 deck and making one move, to get `2 7 9 5 1 6 8 3 4`.

In general, we can generate all size-_N_ decks with length >= _K_ by:
1) pruning the search only when the maximum possible length is less than _K_, and
2) taking each deck with length _X_ > _K_ and making _X_ - _K_ forward moves.

Then on each of these decks we can perform a backward search to find the longest size-(_N+1_) deck which could produce it. Assuming those backward searches don't find anything which matches the already-known lower bound for _L(N+1)_, then when doing the search for size _N+1_ we can use _K_ as the bound for our pruning when there is one fixed card at the end of the deck. Since the execution time is proportional to at least _N!_ (and in fact more, because larger decks take more moves), it's worth spending a lot more effort at one size to save on the next.

We can find the smallest useful _K_ by taking the best-known deck for size _N+1_ and running it forward until _N+1_ is in the last place. For example, when playing the optimal 10-38 deck, 10 ends up in last place after 18 moves. The remaining 9 cards take 20 moves to complete, thus the smallest useful _K_ is 21; looking any deeper than that will not allow us to improve the pruning for size 10.

In principle we could take this further -- we could do the size-(_N-2_) search, appending two cards to each found deck and doing the backward search, to tighten the bound used at size-_N_ when there are two fixed cards at the end of the deck. But by experimentation I found that the large majority of pruning occurs with a single fixed card, and that when there are more fixed cards we almost always prune anyway using the existing bounds, so I didn't take it beyond this first step.

### A few tricks

This search is still going to take a long time, so we can tune the code a bit, taking advantage of the fact that we don't expect to be dealing with decks larger than about 21 cards:
- Use a bit vector to track which cards have already been used. Then we can find the lowest unused card (on most CPUs) with a bit-scanning instruction like that used by `__builtin_ctz()`, and clear the lowest set bit with `x &= (x-1)`.
- If we select the _N_ as the value for the top card, then we know that in one move we will have _N_ in the last position, and we can perform our pruning check right away, before playing out all the subsequent moves.
- The simplest code to perform a top-swapping move is just a loop which swaps cards from the outsides to the middle. But this is terrible for the branch predictor because the loop count is totally unpredictable. Simply creating a big ugly switch statement with 21 copies of the same loop gave a 15% performance boost, and cut mispredictions nearly in half. (Which I don't totally understand, because that creates a branch table, which should have an unpredictable indirect branch to index into it... maybe something about branch history.) We can get another 5-10% by using bespoke combinations byte moves and `__builtin_bswap` for each size. And putting `__builtin_unreachable()` as the default case helps, by telling the compiler it doesn't need to check for sizes not specified in the switch statement.

There is a clever way to represent the deck as a doubly-linked list, described by [Tom Rokicki](https://tomas.rokicki.com/pancake/), where the stored values are the XOR or sum of adjacent cards. This allows a section of the deck to be reversed by updating just two elements, but requires a linked-list traversal to figure out their positions. This can be a huge savings when operating on large decks, but I didn't try this because for small decks my bswap code was so short that it seemed hard to beat.

### Putting it together

_L(N)_ was known up to _N_=19. _N_=20 seemed doable, and maybe _N_=21 at a stretch -- likely to be around 500 times the effort of _N_=19. So my focus was on making the _N_=21 search feasible.

Doing the basic size-19 search took about 670 CPU-days and confirmed the published result that _L(19)_=221. Playing topswops with the best-known 20-card deck puts 20 in the last position after 60 moves, leaving a 19-189 deck. So So then I reran the size-19 search with _K_=189, finding all solutions of length 189 and above.

Then I reran the size-19 search to depth 

## Results

can generate all size-N decks of length at least _L(N)_ - _D_

reduce our pruning threshold by some depth _D_, and also take

we can search size-N decks to some depth D (meaning we consider decks of length at least _L(N)_ - _D_) by reducing our pruning threshold by _D_, and also taking each deck with length _L_ > _D_ 

will only look for deranged permutations

which are one less than the maximum length, we find that there is a single length-29 deck (`6 1 5 9 2 7 8 3 4`), which can also only go backward 2 steps when we append a 10. So we can tighten the bound for the size-10 search to _L(9)_-2=28.
