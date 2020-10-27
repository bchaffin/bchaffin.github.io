---
layout: post
title:  "Repunit Primes"
date:   2020-01-08
---

A [*repunit*](https://en.wikipedia.org/wiki/Repunit) is a number which
is all 1's. Repunit *n*, written as R(n), has *n* 1's, so for example
R(5) = 11111. (Here we are talking about base 10 only.) You could
argue that they're pretty much arbitrary numbers which just happen to
have a nice representation in our favorite base... but I guess that's
a good enough reason for me.

One interesting question is which repunits are prime. I believe that
it is not known whether there are infinitely many prime repunits,
based on this fun [Numberphile
video](https://www.youtube.com/watch?v=eeoBCS7IEqs) in which James
Maynard discusses a proof that there are infinitely many primes which
do not use one base-10 digit, but says that banning two or more digits
is too hard.

Lots of other people have worked on finding repunit primes and have
written a lot about it -- see the [references](#references) below for
the sources I found useful.

## Some fun background math

Another way of writing R(n) is (10<sup>n</sup>-1)/9. If *n* is
composite, then we can use some basic algebra to factor
10<sup>n</sup>-1. If *n* == *a\*b*, then 10<sup>n</sup>-1 is
divisible by 10<sup>a</sup>-1 and 10<sup>b</sup>-1. For example,
R(9)=111111111 is divisible by R(3)=111, because 3 divides 9. This
means that in our search for prime repunits, we only need to consider
numbers with a prime number of 1's.

The next fact that helps our search is that if R(n) is composite, then
its factors will all be of the form *2kn+1*, where *k* is an integer
greater than or equal to 1. So if we want to do some trial division to
see if we can quickly find a factor, we only have to try *2kn+1* for
various values of *k*. And since we really only care about prime
factors (because if there's a composite factor, there must be a
smaller prime factor that would be easier to find), we only need to
try values of *2kn+1* which are prime... which is an interesting
problem (see below).

Finally, as explained in chapter 11 of Beiler's [Recreations in the
Theory of
Numbers](http://www.plouffe.fr/simon/Phys%20et%20Math/Albert%20Beiler%20-%20Recreations%20In%20The%20Theory%20Of%20Numbers.pdf),
a prime can only divide 10<sup>n</sup>-1 if its remainder mod 40 is
one of 8 values: 1, 3, 9, 13, 27, 31, 37, or 39. Primes in general
have 16 possible remainders mod 40, so this eliminates half of all
primes from consideration as possible factors of a repunit.

## Fast trial division

The first step in trying to figure out whether a given repunit is
prime is to try dividing it by some small factors -- we may get lucky
and find that it's composite very quickly.

This presents two problems: how to efficiently select possible factors
to try, and then how to quickly check whether they divide a very large
number (the repunits we will be dealing with are typically in the
millions of digits).

### Generating possible factors

As described above, when doing trial division of R(n), we would like
to try only prime numbers of the form _2kn+1_. Of course computing
_2kn+1_ for each _k_ is easy, but generating only ones which are prime
is hard. Even a single iteration of a Miller-Rabin primality test is
quite expensive, actually more expensive than just checking whether
the number divides R(n). But we don't need to be perfect -- checking
some composite numbers doesn't hurt anything, it's just wasted
time. So we want some way to avoid as many composites as possible,
with very little overhead.

One thing we can use is that as we test successive values of _k_, we
are creating an arithmetic series of candidate factors with a stride
of _2n_. And because _n_ is a prime, we know that this stride is
relatively prime with every other odd prime, which means that as far
as divisibility by small primes goes, it behaves just like sequential
numbers: every 3rd number will be divisible by 3, every 5th number
will be divisible by 5, and so on. So the fact that our numbers are
_2kn+1_ only matters in our initial setup, and otherwise the methods
for avoiding small primes work just as they would for sequential
numbers.

I came up with three approaches, described below. I feel like this is
a problem which must have been solved before, and surely in a better
way -- so if you have a better answer, please tell me!

#### Method #1: Pattern of deltas

We can easily avoid numbers divisible by 3 by just skipping every
third number. We can avoid numbers divisible by 3 or 5 by skipping,
within each block of 15 numbers, the 3rd, 5th, 6th, 9th, 10th, 12th,
and 15th. It's faster and more compact to represent the pattern as a
series of deltas that you add to get from one number to the next --
this is efficient because it allows us to skip directly from one legal
candidate to the next without testing all the numbers in between.

This can be extended to as many factors as you like, but the length of
the pattern that it generates is proportional to the product of all
the primes we are skipping multiples of. Covering the primes up
through 31 is about the practical limit, and even that uses an array
so large that it's going to have some bad cache effects.

#### Method #2: Packed vectors of counters

Say we want to avoid multiples of a 5-bit prime like 23. We can create
a 6-bit counter and initialize it to 32-23. Now we increment the
counter each time we increment _k_, and when it reaches 32 -- which we
can easily determine by looking at the most significant bit -- we know
that we are at a number divisible by 23. We can skip this _k_, and
reset the counter to 32-23.

Then, we can pack lots of those counters into a 64-bit integer. We can
check whether any of them overflows by ANDing with a mask of all the
MSBs of each counter, and then with a bit of shifting and masking we
can reset only those counters which overflowed.

This method has the disadvantage that you have to do the work of
maintaining the counters for every _k_, even the ones that you end up
skipping. But being able to detect multiples of 7 or 8 primes all in
parallel is a big advantage.

#### Method #3: Shift registers

Conceptually, the shift register for prime P is a P-bit rotator with
one set bit. Each time we increment _k_ by _x_, we rotate the register
right by _x_ bits. When the LSB is set, then the current _2kn+1_ value
is divisible by this prime. Mostly there is one register per prime,
but 3\*7 and 5\*11 are small enough to fit in 64 bits, so for those two
pairs we can use one register to cover two primes.

The advantage of shift registers is that we can OR them all together
and then count the number of LSB ones to see how many upcoming numbers
will be divisible by some prime. Then we can just increment _k_ by
that, and skip over them all at once, instead of testing each _k_
individually. The vector counters, by contrast, can tell us whether
this _k_ is a small-prime multiple, but don't give an easy way of
simultaneously checking _k+1_.

#### Which is best?

Despite the advantage of being able to skip over multiple numbers at
once, the shift registers had too much overhead, and don't scale well
to primes greater than 64. So that idea died.

I ended up with a hybrid between the other two. I use a delta pattern
to avoid multiples of 3, 5, 7, 11, and 13, as well as primes which
have an illegal remainder mod 40 -- this is a nice short pattern that
fits in the L1 cache, but it gets most of the benefit of skipping from
one possible _k_ to the next without testing the ones in between. Then
five 64-bit vectors of counters cover primes from 17 up to 197.

(Too Much Information: Combining these methods is not seamless. The
vector counters get a little more complicated because we are
incrementing by more than 1 -- we may skip right past a multiple of
the prime, so we need to reset the counter but not skip this
number. And sometimes we may want to increment _k_ by more than the
width of the smallest counter, which the counters can't really handle;
so the pattern never jumps more than 17, and sometimes allows numbers
which are illegal mod 40 to slip through. These are caught by a final
check.)

The combination of skipping anything divisible by a prime <= 197 and
anything illegal mod 40 rules out nearly 90% of possible _k_
values. Even so, more than 75% of the numbers which make it through
this filter are not, in fact, prime -- but any more expensive check to
remove the composites ends up hurting overall performance. Again, I'm
sure there is a better answer out there somewhere...

### Checking for divisibility

Once a number has made it through the filtering above, we need to
check whether it divides R(n). Instead of actually working with R(n),
which might be a gigantic multi-million-digit number, we can take
advantage of the fact that if P divides R(n), then P also divides
10<sup>n</sup>-1, which means that 10<sup>n</sup> mod P == 1. Modular
exponentiation is quite fast, and never needs to work with numbers
much larger than the modulus, which in this case is a 64-bit number.

This part I borrowed from the [GMP](https://gmplib.org/) function
`mpz_powm`. It has a clever algorithm which repeatedly squares the
base and then computes the remainder mod the modulus using inverse
multiplication, to avoid division. It's logarithmic on the size of the
exponent, which means that it can compute 10<sup>1000000</sup> mod P
in about 20 steps.

I stripped down the code to remove all the cases which
dealt with base/exponent/modulus sizes outside what I needed for this
problem, removed most pointers, arrays, etc., and used built-in
64x64=128 multiplication for the modular reduction. This roughly
doubled the performance compared with just using the library function
directly.

### Performance

On my Skylake system at 4.5GHz, this process can check whether
R(4500007) is divisible by _2kn+1_ for 1 <= k <= 1,000,000,000 in
under 14 seconds. I was pretty happy with that.

There is probably room for improvement -- it might be faster to
compute the remainder mod the product of two factors, and then test
that remainder for divisibility, or maybe to try to hand-vectorize the
code. But small factors are more common than large ones, and so the
benefit of increased trial division is pretty small. In practice,
trial division successfully finds a factor for about 60% of repunits
in my current setup.

## The big guns

Trial division is always worth a shot since it often finds a small
factor. But among repunits with millions of digits, some will not have
any factor small enough to be in range of trial factoring. For that we
need a different approach.

[Primality testing](https://en.wikipedia.org/wiki/Primality_test) is a
huge field, and there are much better places to learn about it than
here. There are several tests which can say that a number is either
definitely composite, or probably prime with some chance of being
wrong; by running multiple independent tests we can make the
probability of a wrong answer arbitrarily low.

The [Fermat test](https://en.wikipedia.org/wiki/Fermat_primality_test)
is (as far as I know) the simplest and fastest way to do a single
check of whether a number is prime. Fermat's Little Theorem says that:

`a^(p-1) mod p == 1`

if _p_ is prime and _a_ is not a multiple of _p_. So we just pick some
value of _a_, say 3, and compute 3^(R(n)-1) mod R(n). If the result is
not 1, then R(n) can't be prime. Which is nice and simple... except
R(n) is a huge number.

Enter [OpenPFGW](https://sourceforge.net/projects/openpfgw/), a great
tool which uses the GWNum library from
[prime95](https://www.mersenne.org/download/) to do a Fermat test with
super-optimized FFT multiplication. Doing a modular exponentiation
with a 4 million-digit modulus takes a long time -- several hours --
but it virtually always gives a guarantee that a number is composite.

## Putting it all together

The best way of using PFGW depends a lot on what size numbers you're
testing. For small numbers, the FFT data fits in the core caches and
you can run one instance on every core. But larger numbers don't fit,
and running too many at once just thrashes the cache and slows
everything down.

After a lot of experimentation, for the size numbers I'm interested in
(around 4 million digits), I settled on running four 4-threaded
instances of PFGW on 16 cores. Four 8-threaded instances gives
slightly higher performance, but with 6 or more threads there is some
resonance in the power draw, probably from threads rapidly starting
and stopping high-power vector computations. It makes an irritating,
high-pitched scritching sound, and is probably also producing some
kind of mechanical stress somewhere, which seems bad. So four threads
it is. One other core runs two threads of trial factoring. 

A script ties them all together. It reads a list of R(n) values to be
checked, maintains a pool of in-flight numbers which rotate through
trial factoring, and as each PFGW job completes it takes the number
from the pool which has had the most trial factoring and hands it off
to PFGW.

## Update, October 2020

I got a bit of an education from Danilo Nitsche of the [Repunit Primes
Project](https://kurtbeschorner.de/). Trial factoring of repunits is
much more efficiently done with a GPU, using an algorthm roughly
similar to what's described above, but massively parallel. Everything
under R(5000000) has been trial factored far beyond what can be done
with a general-purpose core.

Also, [prime95](https://www.mersenne.org/download/) is substantially
faster than PFGW for probable prime checks. Running 2 four-threaded
workers and 3 three-threaded workers (one thread per core) gives the
best throughput -- about 30% more than with PFGW.

The syntax for prime95's worktodo.txt file is not well documented, but
thanks to Danilo here it is:

    [Worker #1]
    PRP=1,10,4371223,-1,99,0,3,1,"9"


## Results

[A004023](https://oeis.org/A004023) gives the indices of the repunits
which are known to be prime or probably prime. The largest currently
known is R(270343), which was found in 2007.

The [Repunit Primes
Project](https://kurtbeschorner.de/) has checked (as of October 2020)
up a little past R(4375000). Before learning that I started at
R(4500000), with results below.

#### Completed searches

[R(4500000) - R(4600000)](repunits-4500000-4600000.txt) -- no primes

## References

General repunit background:
- [Repunit Wikipedia entry](https://en.wikipedia.org/wiki/Repunit)
- [Repunit in Prime Glossary](https://primes.utm.edu/glossary/page.php?sort=Repunit)
- [Repunits at Mathworld](http://mathworld.wolfram.com/Repunit.html)
- [Recreations in the Theory of Numbers, chapter 11](http://www.plouffe.fr/simon/Phys%20et%20Math/Albert%20Beiler%20-%20Recreations%20In%20The%20Theory%20Of%20Numbers.pdf)

Factoring repunits:
- [Huge database of known factors of repunits](https://stdkmd.net/nrr/repunit/)
- [Factoring repunits](https://www.jstor.org/stable/2321382?read-now=1&seq=1) - W. M. Snyder
- [The Mystique of Repunits](https://www.jstor.org/stable/2689643?read-now=1&seq=1) - S. Yates
- [Fun with repunit divisors](https://mathlesstraveled.com/2011/11/11/fun-with-repunit-divisors/)
- [Repunit factors on World of Numbers](http://www.worldofnumbers.com/repunits.htm)
- [Factoring repunits](https://gmplib.org/~tege/repunit.html) - including C source code

Repunit primes:
- [Repunit R49081 is a probable prime](https://www.ams.org/journals/mcom/2002-71-238/S0025-5718-01-01319-9/S0025-5718-01-01319-9.pdf)
- [A004023](https://oeis.org/A004023) - indices of known prime/PRP repunits
