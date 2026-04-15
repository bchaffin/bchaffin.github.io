---
layout: post
title:  "Primes of the form 3^k - 2^k"
date:   2026-04-14
---


### The sequence

The [OEIS](https://oeis.org) sequence [A057468](https://oeis.org/A057468) gives the values of _k_ for which 3<sup>k</sup>-2<sup>k</sup> is prime (or probably prime, for larger numbers). The largest known term is 1059503, meaning 3<sup>1059503</sup> - 2<sup>1059503</sup>, which is a number with 505512 digits, is probably prime. Finding the next term seemed like an interesting challenge.

### First steps

As noted in [A001047](https://oeis.org/A001047), 3<sup>k</sup>-2<sup>k</sup> can only be prime if _k_ is prime, which immediately reduces the number of exponents we have to check to just the primes. 

The first step is trial division to look for small divisors. After testing many numbers for divisibility by a long list of small primes, I noticed an interesting pattern: **if 3<sup>k</sup>-2<sup>k</sup> is composite, all factors appear to be of the form 2n*k+1, for some _n_ >= 1**. I think this derives from the fact (also noted in A001047) that _p_ divides 3<sup>_p_</sup>-2<sup>_p_</sup>-1 if _p_ is prime. But since trial division is in some sense just an optional optimization, I just used this observation without trying to prove it.

So instead of doing trial division by all small primes, we only need to look at even multiples of *k* plus 1, which allows us to do trial division up to a much larger threshold. Of course not all 2n\*k+1 are prime, and we really only want to test divisibility by primes. But doing a full primality test on a 64-bit number (with GMP's `mpz_probab_prime_p()`) is actually slower than just checking whether it divides 3<sup>k</sup>-2<sup>k</sup>. Instead we can filter out most non-prime candidates by doing a little trial division on our trial division: for each 2n\*k+1 we check whether it is divisible by a list of small primes, and if not then we check whether it divides 3<sup>k</sup>-2<sup>k</sup>.

The mini-trial division led me to this nice trick for divisibility of unsigned 64-bit numbers, gleaned from gcc's assembly output:
```
    // x and modulus are uint64_t
    uint64_t inverse = ((__uint128_t)1 << 64) / modulus;
    if (x * -inverse < inverse)
        // x % modulus == 0
```

To check whether *m* divides 3<sup>k</sup>-2<sup>k</sup>, it was faster to compute 3<sup>k</sup> mod *m* and 2<sup>k</sup> mod *m* (using `mpz_powm`), and check whether they are equal.

### The Fermat test

The [Fermat test](https://en.wikipedia.org/wiki/Fermat_primality_test) is based on Fermat's Little Theorem, which says that:

`a^(p-1) mod p == 1`

if _p_ is prime and _a_ is not a multiple of _p_. So for numbers _n_ which pass the trial factoring, we compute 2<sup>n-1</sup> mod _n_; if the result is not 1, then this number is composite.

This modular exponentiation can take quite a while (since the exponent is in the 500k - 2 million digit range), and I wanted to keep my batch jobs short. At first I looked for a multi-threaded modmul function, which led me to the wonderful [FLINT library](https://flintlib.org/), which is a great discovery. This helped, but it can only use a limited number of threads and eventually even multi-threaded jobs were taking too long. So I extracted the code from FLINT's `fmpz_powm()` and added the ability to pass in a starting and ending exponent, and save the internal state to disk. This let me break up long modmul operations into many pieces.

### Results

With these techniques, I was able to test all _k_ up to 4,000,000; at the top end of that range 3<sup>k</sup>-2<sup>k</sup> has about 1.9 million digits. Unfortunately, all of them were found to be composite. Complete results are in these files:
- [1059503 to 2000000](3k2k-results-1m2m.txt)
- [2000000 to 3000000](3k2k-results-2m3m.txt)
- [3000000 to 4000000](3k2k-results-3m4m.txt)

A random odd number _R_ has roughly `2/ln(R)` chance of being prime. Assuming that 3<sup>k</sup>-2<sup>k</sup> (for prime _k_) has a similar likelihood of being prime -- which may not be a valid assumption! -- then there is only about a 2.6% chance of finding a prime for _k_ between 4,000,000 and 5,000,000.
