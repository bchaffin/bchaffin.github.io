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

The number of steps in the game for a given deck is the _length_ of the deck. The question, of course, is what is the maximum length of a deck of size _N_. It's known that there is a [quadratic lower bound](https://doi.org/10.1016/j.tcs.2010.08.011), and an exponential upper bound of the (_N_+1)th Fibonacci number (proof in the Wikipedia article). [OEIS](https://oeis.org/) sequence [A000375](https://oeis.org/A000375) gives the maximum length of the game for various deck sizes. It was most recently [extended to sizes 18 and 19](https://arxiv.org/abs/2103.08346) in 2021. Topswops was also the subject of an [AZsPCs programming contest](http://azspcs.com/Contest/Cards) in 2011, which produced longest-known decks for prime sizes up to 97.

## A Topswops search algorithm

Knuth gives several algorithms for finding a longest Topswops deck in Volume 4, section 7.2.1.2, solution to exercise 108. The last one, which he refers to as the "better" algorithm, is a forward search which fills in the value of cards only when it becomes necessary. We start with a deck of blank cards; then we choose a number for the topmost card and make topswops moves until the top card is again blank. By trying each value not already used for blank cards, we can recursively search all permutations while simultaneously making the topswop moves, so that we don't have to repeat the same initial moves for similar decks. And Knuth has a clever trick for keeping track of where in the initial deck each card started, so that when we choose a value for the top card we know which original card we're choosing: we label the cards in the initial deck with the negative of their position, so the starting deck is -1, -2, -3, etc. Then a blank card is anything negative, and when we choose a value for it we also keep track of which card we assigned in a separate initial deck.

We can prune the search in a few ways, to avoid considering all _N_! decks:

- Never choose 1 as the top card until all other values have been used. Choosing 1 ends the game, so if there is any other choice available then obviously it will result in a longer deck.
- A longest deck must be a _derangement_; that is, it must not have any cards in their "home" position, i.e. card _M_ in the _M_th position. If it did, we could do a backward move by swapping the top _M_ cards, giving a deck whose length is 1 greater than the current deck.
- 
