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

The question, of course, is how many steps the game can take for a deck of size _N_. It's known that there is a [quadratic lower bound](https://doi.org/10.1016/j.tcs.2010.08.011), and an exponential upper bound of the (_N_+1)th Fibonacci number (proof in the Wikipedia article). [OEIS](https://oeis.org/) sequence [A000375](https://oeis.org/A000375) gives the maximum length of the game. It was most recently [extended to sizes 18 and 19](https://arxiv.org/abs/2103.08346) in 2021. Topswops was also the subject of an [AZsPCs programming contest](http://azspcs.com/Contest/Cards) in 2011, which produced longest-known decks for prime sizes up to 97.

## A Topswops search algorithm

Knuth gives several algorithms for finding a longest Topswops deck to Volume 4, section 7.2.1.2, exercise 108. 
