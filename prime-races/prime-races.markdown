---
layout: post
title:  "Prime Races"
date:   2026-05-30
---


## Background

Thanks to [Mike Keith](https://www.cadaeic.net/) for introducing me to this topic.

The [Chebyshev bias](https://en.wikipedia.org/wiki/Chebyshev%27s_bias) is the observation that if one counts primes mod 4 up to some limit, there are very often more primes of the form 4k+3 than 4k+1. In general, primes mod *M* are "lumpily" distributed, with some residues trailing others for long (sometimes extremely long) stretches; however Littlewood proved that every residue will lead at some point, and in fact will lead an infinite number of times. There's a good overview in this [excellent paper](https://dms.umontreal.ca/~andrew/PDF/PrimeRace.pdf) by Andrew Granville and Greg Martin.

Mike's initial question was about primes mod 100. There are 40 possible residues, and 39 of them are known to lead at some point; the last of those is 43, which takes the lead at 11342219743. Only 41 has never been in the lead, so naturally we'd like to find the place where it takes the lead (which it must, eventually).

Some other races we'll look at:
- [A275939](https://oeis.org/A275939) - For every modulus *q*, consider the race between residue 1 and -1. Residue 1 is generally disadvantaged, so the question is when does 1 take the lead. The answer was known for all *q* < 1000, except for *q*=12 and *q*=24.
- [A380877](https://oeis.org/A380877) - When is residue 1 tied with residue 7, mod 12. Only 8 terms were known, the largest of which was 461.
- [A390417](https://oeis.org/A390417) - For mod 10, there are four possible residues (1, 3, 7, and 9). Look for when each of the 24 possible "leaderboard" rankings occurs. Five of the possible rankings had not been found.
- [A130911](https://oeis.org/A130911) - Race between evil primes (which have an even number of 1s in their binary representation) and odious primes (which have an odd number of 1s). It is conjectured that evil never leads after the first 3 primes.
- [A156549](https://oeis.org/A156549) - Race between primes which have an even number of 0s in binary vs. those which have an odd number of 0s. It appears that an odd number of 0s always leads. It's somewhat surprising that odd leads for both 1s and 0s; more on this below.

## Computation

I look at these races for all primes up to 10<sup>19</sup>. The basic code for counting a prime race is quite simple: using the super-fast [primesieve](https://github.com/kimwalisch/primesieve) library, generate sequential primes, and count up their residues mod some value. But to make real progress we want to parallelize this process; while counting residues within some range, we don't know the full counts and so we can't tell whether one residue takes the lead.

To see the solution, we can take the mod-100 race as an example. We are interested only in residue 41, so as we process each block we track the largest lead that 41 ever has over each of the other residues, and output that at the end of the block, along with the counts for each residue. Then in a post-processing step we look at the output of each block sequentially, calculating the total count for each residue, and checking whether 41 got far enough ahead within this block that it could have taken the lead.

For example, say that at the beginning of this block, 41 was behind residue R by 1000. If the largest lead that 41 had over R at any point within this block is less than or equal to 1000, then it can't possibly pass R overall; on the other hand, if the largest lead within this block is greater than 1000, then 41 will pass R at some point. If 41 passes *all* of the other residues, then it's possible that it takes the overall lead -- though not guaranteed, because even if it is ahead of every other residue at some point, it may not be ahead of all of them at the same time.

If we find a block where 41 passes all the other residues, we'll have to rerun that block in "safe mode", starting with the full counts up to that point, to see if it really does come out ahead.

For simpler 2-way races, like the residue +/-1 race, there's no doubt -- if 1's largest lead is greater than its starting deficit, then it takes the lead within this block. But we'll still need to rerun in safe mode to find exactly where.

For the mod-10 "leaderboard" race, we track the largest lead of every residue over every other residue (4*3 = 12 values total), so that we can check whether, within each block, first place is ahead of second, third, and fourth; second is ahead of third and fourth; and third is ahead of fourth. Again, we'll need to rerun to check whether this leaderboard really happens, and find exactly where.

## Results

### Mod-100 race
Sadly, residue 41 never takes the lead up to 10^19. Here's a graph which shows, at the end of each block of 2.5*10<sup>12</sup>, how far 41 is behind the leader, and which residue is in the lead. (Note that this does not show every change of leader within the blocks, or the closest that 41 ever comes to the lead; but it gives a good idea of the overall behavior.)

## Prime tabulation

As long as I was generating primes over a large range, I thought I might as well collect some other information along the way. Some of the basic counts of primes, such as the number of primes below 10<sup>n</sup>, are already known to large numbers. But I was able to extend a number of other sequences:
- [A095005](https://oeis.org/A095005) and [A095006](https://oeis.org/A095006) - number of odious/evil primes in power-of-2 ranges
- [A129542](https://oeis.org/A129542) and [A129697](https://oeis.org/A129697) - number of isolated (non-twin) primes less than 10<sup>n</sup>, and their sum
- [A033843](https://oeis.org/A033843) - number of twin primes less than 2<sup>n</sup>
- [A007508](https://oeis.org/A007508) and [A118552](https://oeis.org/A118552) - number of twin primes less than 10<sup>n</sup>, and their sum
- [A080840](https://oeis.org/A080840) and [A152127](https://oeis.org/A152127) - number of cousin primes (primes 4 apart) less than 10<sup>n</sup>, and their sum
- [A080841](https://oeis.org/A080841) - number of primes 6 apart less than 10<sup>n</sup>
- [A093738](https://oeis.org/A093738) and [A341843](https://oeis.org/A341843) - number of *consecutive* primes 6 apart less than 10<sup>n</sup>, or less than 2<sup>n</sup>
- [A091644](https://oeis.org/A091644) and friends - number of primes less than 10<sup>n</sup> containing digit *X*
- [A091634](https://oeis.org/A091634) and friends - number of primes less than 10<sup>n</sup> which do *not* contain digit *X*
- [A231590](https://oeis.org/A231590) and friends - total number of digit *X* in primes less than 10<sup>n</sup>
- [A244191](https://oeis.org/A244191) and [A244265](https://oeis.org/A244265) - most common final digit for primes less than 10<sup>n</sup>, and its frequency
- [A091117](https://oeis.org/A091117) - number of primes which are 2 mod 5 less than 10<sup>n</sup>
- [A091115](https://oeis.org/A091115) and friends - mod-6 residue counts less than 10<sup>n</sup>
- [A091120](https://oeis.org/A091120) and friends - mod-7 residue counts less than 10<sup>n</sup>
- [A091126](https://oeis.org/A091126) and friends - mod-8 residue counts less than 10<sup>n</sup>
- [A073506](https://oeis.org/A073506) and friends - mod-10 residue counts less than 10<sup>n</sup>
- [A091161](https://oeis.org/A091161) and friends - mod-12 residue counts less than 10<sup>n</sup>
- [A091165](https://oeis.org/A091165) and friends - mod-30 residue counts less than 10<sup>n</sup>
