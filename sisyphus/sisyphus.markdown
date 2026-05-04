
---
layout: post
title:  "The Sisyphus Sequences"
date:   2026-04-01
---


### The sequence

The Sisyphus sequences are defined by simple iterative rules: if the previous term was even, we divide by two; if the previous term was odd, then we add something. For [A350877](https://oeis.org/A350877), which actually bears the Sisyphus name in the OEIS, we add the next prime which has not yet been added. For [A347297](https://oeis.org/A347297), which I call the "simple Sysiphus sequence", we add the next integer which has not yet been added.

The resulting behavior is interesting, as the sequence rises up by ever-larger steps before falling back down again as all the powers of 2 are divided out. The OEIS entry has some nice scatterplots showing the patterns created. It's conjectured that every number will eventually appear, and to get a feel for whether that's likely to be true we can look for record-low values, when we see the lowest number which has not previously appeared in the sequence.

### Computation - simple version

The simple sequence is straightforward to compute. We can speed things up by observing that because we only add when the previous term was odd and the addition steps are adding sequential numbers, we will always add an even number, then add an odd number, then divide by 2 some number of times. So our main loop can always do "add, add, divide", and we can combine the additions by just tracking a single value which is `next_i + next_i + 1`. To do the division, we can use `__builtin_ctzl()` to count the number of trailing zeros (powers of 2) and then shift by that amount. The final inner loop is totally branchless and very small:
```
// Start with val odd and next_i even so we know we will always add twice and then divide
while (1) {
    // We can do both additions at once, and track only the value of (nexti + nexti+1)
    val += nextinc;
    nextinc += 4;

    // Result is even, so shift
    int zeros = __builtin_ctzl(val);
    val >>= zeros;

    steps += zeros + 2; // Two addition steps and z1 divisions
}
```

On my home machine this computes 10 billion terms every 3 seconds, or about one term every 1.3 clock cycles (the full version has some additional checks for small values and printing status updates).

### Computation - prime version

The prime-addition sequence uses very similar code, except that since we are always adding an odd number we will always add just once and then divide. To generate the primes, I use the amazing [primesieve library](https://github.com/kimwalisch/primesieve). I create a pool of shared memory regions using `mmap` with the `MAP_ANONYMOUS | MAP_SHARED` flags, and spawn a pool of threads which is each responsible for generating primes within a region and writing them into one of the shared buffers. The main program waits for the next thread in line to finish, consumes all of the primes in its buffer, and then launches a new thread to refill it. 

As long as there are enough prime-generating threads in the pool, this allows the main thread to compute terms of the sequence at full speed. On my machine it takes about 13 generators to keep the main thread fed, and it runs at about half the speed of the simple version. It's slower partly because it only does two terms per loop, and partly because it has to stream primes from memory instead of just using a simple increment. But it's still computing one term every 2.5-3 cycles, comsuming ~550 million primes (~4.5GB of data) per second.

### Results
