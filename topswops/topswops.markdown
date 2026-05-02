---
layout: post
title:  "Longest Topswops decks for sizes 20 and 21"
date:   2026-04-14
---


## The Topswops game

[Topswops](https://en.wikipedia.org/wiki/Topswops) is a game invented by John Conway. The procedure is simple: start with a deck of _n_ cards, numbered 1 through _n_, shuffled. Look at the number of the card on top, then reverse the order of that many cards on the top of the deck. Continue until the top card is a 1. For example, say we start with the 5-card deck **3, 1, 4, 5, 2**. Then the game goes as follows, finishing after 7 moves:

- move 0: 3 1 4 5 2
- move 1: <span style="color:red">4 1 3</span> 5 2
- move 2: <span style="color:red">5 3 1 4</span> 2
- move 3: <span style="color:red">2 4 1 3 5</span>
- move 4: <span style="color:red">4 2</span> 1 3 5
- move 5: <span style="color:red">3 1 2 4</span> 5
- move 6: <span style="color:red">2 1 3</span> 4 5
- move 7: <span style="color:red">1 2</span> 3 4 5

The number of steps in the game for a given deck is the _length_ of the deck. The question, of course, is what is the maximum length of a deck of size _n_, which Knuth calls _f(n)_. It's known that there is a [quadratic lower bound](https://doi.org/10.1016/j.tcs.2010.08.011), and an exponential upper bound of the (_n+1_)th Fibonacci number (proof in the Wikipedia article). OEIS sequence [A000375](https://oeis.org/A000375) gives the maximum length of the game for various deck sizes. It was most recently [extended to sizes 18 and 19](https://arxiv.org/abs/2103.08346) in 2021. Topswops was also the subject of an [AZsPCs programming contest](http://azspcs.com/Contest/Cards) in 2011, which produced longest-known decks for prime sizes up to 97.

In the discussion below, I'll use "an _n-l deck_" to mean an _n_-card deck that takes _l_ moves; the example above is a 5-7 deck.

## Results

But let's not bury the lede -- the new results are:

#### _f(20)_ = 249, with a unique length-249 deck: `2 8 11 6 19 17 13 7 18 3 9 20 15 1 12 4 10 14 5 16`

#### _f(21)_ = 282, with five decks tied for maximum length:
```
3 4 10 20 7 21 19 15 14 8 16 13 5 2 9 12 18 17 1 11 6
3 17 7 13 10 14 6 19 15 21 1 9 2 16 18 20 11 8 5 4 12
6 11 20 18 16 9 14 10 13 2 1 21 15 8 17 19 7 3 5 4 12
7 17 8 13 10 14 6 19 15 21 1 9 2 16 18 20 11 3 5 4 12
14 10 13 8 17 11 20 18 16 2 9 1 21 15 6 19 7 3 5 4 12
```

At least some of these decks had been previously found through optimization searches, but were not kown for sure to be optimal or complete.

## A Topswops search algorithm

Knuth gives several algorithms for finding a longest Topswops deck in Volume 4, section 7.2.1.2, solution to exercise 107. The last one, which he refers to as the "better" algorithm, is a forward search which fills in the value of cards only when it becomes necessary. We start with a deck of blank cards; then we choose a number for the topmost card and make topswops moves until another blank card is on top. By trying each value not already used for blank cards, we can recursively search all permutations while simultaneously making the topswop moves, so that we don't have to repeat the same initial moves for similar decks. And Knuth has a clever trick for keeping track of where in the initial deck each card started: we label the cards in the initial deck with the negative of their position, so the starting deck is -1, -2, -3, etc. Then a blank card is anything negative, and when we choose a value for it we also separately keep track of which card we assigned in the initial deck.

### Pruning the search

We can prune the search in a few ways, to avoid considering all _n_! decks. The first two pruning strategies are straightforward:

- Never choose 1 as the top card until all other values have been used. Choosing 1 ends the game, so if there is any other choice available then obviously it will result in a longer deck.
- A longest deck must be a _derangement_; that is, it must not have any cards in their "home" position, i.e. card _m_ in the _m_-th position. If it did, we could do a backward move by flipping the top _m_ cards (thus putting _m_ on top of the deck), giving a deck whose length is 1 greater than the current deck.

The third pruning strategy gets more involved. Say we are searching for the best 10-card deck. At some point, maybe card 10 ends up in the last position. Now it can never move again, and effectively we continue the game with a 9-card deck. But if we already know _f(9)_, the maximum length of a 9-card deck, then that is an upper bound for the number of moves remaining; and if that upper bound plus the number of moves so far is not more than the current best 10-card deck, then we can stop searching right away, without filling in any remaining blank cards.

Knuth says that once the largest _m_ cards are in order at the end of the deck, we can use _f(n-m)_ as a bound. Others have observed that we can do better: we can apply the same bound if the largest _m_ cards are at the end of the deck _**in any order**_, which is a significant improvement for larger decks.

### A backward search

Knuth's "good" algorithm, which he attributes to Pepperdine, is similar except that it moves backwards. We start with a deck that has 1 on top and the rest of the cards blank. At any point the possible backward moves correspond to any known card which is in its home position, or any blank card, which we can assign the value of its current position; we continue until there are no blank cards and no more backward moves possible.

Although Knuth does not mention it, very similar pruning can be applied to the backward search. A block of cards at the end of the deck is fixed if, for each card:
1) its value is known and it is not in its home position, or
2) it is blank, but the value of its current position has alread been assigned to another card.

With this pruning, the backward algorithm seems to make fewer top-flipping on average than the forward search, and does not require recording the initial deck. However there is a choice to be made after almost every move, because there are usually multiple backward moves available, which requires a lot more saving and restoring of decks (through deeper recursion, or however you choose to implement it). In the forward search, we only make a choice when the top card is blank, so the recursion is only ever _n_-deep. In my implementation, the extra saving and restoring of the deck outweighs the savings from fewer top-flips, and the backward search was generally 30-50% slower than the forward search. But I can't shake the feeling that there is some improvement that would make the backward search outperform the forward search.

### More pruning

But we can do even better than pruning based on _f(n-m)_. Consider the 10-card search again: with the logic above, we would say that once 10 is in the last position, there can be at most _f(9)_=30 moves remaining. But can we do even that well? The longest 9-card deck is unique: `6 1 5 9 7 2 8 3 4`. We can append a 10 to that and then work backwards to see what 10-card deck (or decks) could lead to it, producing this series of decks:
```
6 1 5 9 7 2 8 3 4 10
10 4 3 8 2 7 9 5 1 6
3 4 10 8 2 7 9 5 1 6
```
After 2 moves, no card is in its home position, so no more backward moves are possible. Thus there is a unique 10-32 deck which produces the maximum-length 9-card deck. (Note that, in general, there could be multiple possible backward moves at one time, so we may have to do a real recursive search to find the longest backward chain of moves.) Having done this tiny bit of work, and once we know that _f(10)_ > 32, we can tighten our bound by 1: for a maximum-length 10-card deck, once 10 is in the last position the remaining deck cannot achieve _f(9)_, and there can be at most _f(9)_-1=29 moves remaining.

We can extend this method: during the size-9 search, we can do some extra work and look for 9-29 decks (one shorter than _f(9)_), and do the same backward search on those; if they also cannot achieve a new 10-card best, then we can reduce our bound by 1 again. But there's a pitfall here: our search wants to take advantage of the second pruning technique above and only look for derangements. This works when looking for maximum-length decks, but not when we also want to look for shorter decks. For example, there is exactly one 9-29 deck that our search will not find: the result of starting with the optimal 9-30 deck and making one move, to get `2 7 9 5 1 6 8 3 4`.

In general, we can generate all size-_n_ decks with length >= _k_ by:
1) pruning the search only when the maximum possible length is less than _k_, and
2) taking each deck with length _x_ > _k_ and making _x_ - _k_ forward moves.

Then on each of these decks we can perform a backward search to find the longest size-(_n+1_) deck which could produce it. Assuming those backward searches don't find anything which matches the already-known lower bound for _f(n+1)_, then when doing the search for size _n+1_ we can use _k_ as the bound for our pruning when there is one fixed card at the end of the deck. Since the execution time is proportional to at least _n!_ (and in fact more, because larger decks take more moves), it's worth spending a lot more effort at one size to save on the next.

We can find the smallest useful _k_ by taking the best-known deck for size _n+1_ and running it forward until _n+1_ is in the last place. For example, when playing the optimal 10-38 deck, 10 ends up in last place after 18 moves. The remaining 9 cards take 20 moves to complete, thus the smallest useful _k_ is 21; looking any deeper than that will not allow us to improve the pruning for size 10.

In principle we could take this further -- we could do the size-(_n-2_) search, appending two cards to each found deck and doing the backward search, to tighten the bound used at size-_n_ when there are two fixed cards at the end of the deck. But by experimentation I found that the large majority of pruning occurs with a single fixed card, and that when there are more fixed cards we almost always prune anyway using the existing bounds, so I didn't take it beyond this first step.

### A few tricks

This search is still going to take a long time, so we can tune the code a bit, taking advantage of the fact that we don't expect to be dealing with decks larger than about 21 cards:
- Use a bit vector to track which cards have already been used. Then we can find the lowest unused card (on most CPUs) with a bit-scanning instruction like that used by `__builtin_ctz()`, and clear the lowest set bit with `x &= (x-1)`.
- If we select _n_ as the value for the top card, then we know that in one move we will have _n_ in the last position, and we can perform our pruning check right away, before playing out all the subsequent moves.
- The simplest code to perform a top-swapping move is just a loop which swaps cards from the outsides to the middle. But this is terrible for the branch predictor because the loop count is totally unpredictable. Simply creating a big ugly switch statement with 21 copies of the same loop gave a 15% performance boost, and cut mispredictions nearly in half. (Which I don't totally understand, because that creates a branch table, which should have an unpredictable indirect branch to index into it... maybe something about branch history.) We can get another 5-10% by using bespoke combinations byte moves and `__builtin_bswap` for each size. And putting `__builtin_unreachable()` as the default case helps, by telling the compiler it doesn't need to check for sizes not specified in the switch statement.

There is a clever way to represent the deck as a doubly-linked list, described by [Tom Rokicki](https://tomas.rokicki.com/pancake/), where the stored values are the XOR or sum of adjacent cards. This allows a section of the deck to be reversed by updating just two elements, but requires a linked-list traversal to figure out their positions. This can be a huge savings when operating on large decks, but I didn't try this because for small decks my bswap code was so short that it seemed hard to beat.

### Putting it together

_f(n)_ was known up to _n_=19. _n_=20 seemed doable, and maybe _n_=21 at a stretch -- likely to be around 500 times the effort of _n_=19. So my focus was on making the _n_=21 search feasible.

Doing the basic size-19 search took about 670 CPU-days and confirmed the published result that _f(19)_=221. Playing topswops with the best-known 20-card deck (length 249) puts 20 in the last position after 60 moves, leaving a 19-189 deck. So then I reran the size-19 search with _k_=189, finding all solutions of length 189 and above, which took about 4 times as long as the original search. This found that a size-19 deck with length >= 190 can only appear in a 20-card deck of length <= 242. So if I just wanted to find the best size-20 dL(eck, then when 20 is in the last position I could prune assuming that the remaining 19 cards cannot take more than 189 moves -- which is 32 less than _f(19)_=221, a huge improvement.

But I'm aiming at _n_=21, so I want to run the size-20 search to some greater depth, and I want to choose that depth to minimize the total time for the size-20 and size-21 searches. I took a sampling of sub-trees across both searches, and measured the effect of various options on the execution time. I settled on running size-20 with _k_=240 (depth of 9), which (the deep size-19 search tells me) can still use 192 as the 19-card pruning threshold, which is 29 less than _f(19)_. This then allows the size-21 search to use 249-9=240 as the pruning threshold, which reduces execution time by about 38%. Going deeper at size 20 required a much higher pruning threshold, greatly increasing the time for the size-20 search, while giving only modest reduction in the size-21 search.

## Detailed results

The size-20 search took ~7200 CPU-days, only ~10x more than the size-19 search due to the stronger pruning, and found that **_f(20)_=249**. The size-21 search took ~XXXXXX CPU-days, and found that **_f(21)_=282**.

Here is a complete list of optimal decks for all sizes up to 21:



