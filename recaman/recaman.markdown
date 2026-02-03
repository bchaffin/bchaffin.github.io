---
layout: post
title:  "The Recamán Sequence"
date:   2024-02-20
---


### The Recamán Sequence

The [Recamán sequence](https://en.wikipedia.org/wiki/Recam%C3%A1n%27s_sequence) is a fascinating integer sequence. It is in the [OEIS](https://oeis.org/) as [A005132](https://oeis.org/A005132); in fact its first terms spiral around the outside of the OEIS logo. It features in the Numberphile video [The Slightly Spooky Recamán Sequence](https://www.youtube.com/watch?v=FGC5TdIiT9U). It has inspired some [beautiful artwork](https://oeis.org/A005132/a005132_1.png), and is discussed in many places on the internet.

I first heard of this sequence in a talk by Neil Sloane in 2006, and I have returned to it many times over the years, through many generations of computing hardware and compilers. Altogether I've probably invested more time and energy in this project than almost any other.

The definition is simple: the first term, a(0), is 0. Then to get the nth term, we subtract n from the previous term, if the result would be non-negative and has not already appeared in the sequence; otherwise, we add n to the previous term. (Duplicate terms are allowed when adding.) So to get the first few terms:
- For a(1), we try subtracting 1 from a(0)=0, but that's negative, so instead we add to get a(1) = 1.
- For a(2), we try subtracting 2 from a(1)=1, but that's negative, so we add to get a(2) = 3.
- For a(3), we try subtracting 3 from a(2)=3, but 0 has already appeared at a(0), so we add to get a(3) = 6.
- For a(4), we try subtracting 4 from a(3)=6, and this time it works: 2 is positive and has not already appeared, so a(4) = 2.

<img src="rec-100.png" width="375"> <img src="rec-1000.png" width="375">
<img src="rec-10000.png" width="375"> <img src="rec-100000.png" width="375">

![Recaman sequence 100 terms](rec-100.png) ![Recaman sequence 1000 terms](rec-1000.png)
![Recaman sequence 10000 terms](rec-10000.png)
![Recaman sequence 100000 terms](rec-100000.png)

TBD: graphs, characterization of overall behavior, questions and goal of computation

### Computing it

Computing terms of the sequence one by one is easy, just following the rule above. But we can go faster: in the graphs above we can see that the sequence tends to alternate addition and subtraction, ping-ponging between two ranges of consecutive numbers. For example, terms a(6) - a(17) are: 13, 20, 12, 21, 11, 22, 10, 23, 9, 24, 8, 25; these form a lower range of 13 down to 8, and an upper range of 20 up to 25. In general, this ping-pong pattern will continue until one of two things happens:

1. The lower range bumps into a number which has already appeared in the sequence, so we can't subtract; then we add twice in a row
2. The lower range passes over a "hole" farther down which allows us to subtract twice in a row

At the beginning of a ping-pong section, we can look ahead to find the which of these two terminal conditions will happen first and in how many steps, and then we know that all the intervening terms form two contiguous ranges, without having to compute them individually. In practice, the ping-pong sections get exponentially longer as the sequence progresses, so the computation also accelerates.

Detecting condition #1 is relatively easy; we just look for the closest number below the lower range which is already taken. Condition #2 is more complicated, because only every third number is tested. Using the same example as above, a(6)=13; to get a(7) we try 13-7=6, which has already appeared. In this case 6 was the potential "hole" which would have allowed us to subtract twice in a row. Now a(8)=12, and to get a(9) we try 12-9=3; 3 is the potential "hole". So to detect condition #2 we need to find the first lower untaken number which is a multiple of 3 steps away.

All that takes a little fiddling to get right, but the real kicker is the part about whether a number has already appeared in the sequence: you have to remember its entire history, and be able to refer back to it constantly! From here on, it's all about how to efficiently store and compute with these vast quantities of data.

### Little big numbers

We're going to be getting into some pretty big numbers. There are plenty of libraries like GMP which handle arbitrary-precision integers, but their numbers are typically implemented as a data structure which has a storage array of 64-bit elements for the data, and some additional fields containing other information about the number, like its size. Also the storage array may be dynamically allocated, which means there is additional metadata maintained by the `malloc` library. The numbers of this sequence are big-ish, but not _that_ big -- up to a couple hundred bytes -- so that overhead is significant. I want to store as many numbers as possible as densely as possible, without any space wasted on metadata or padding out to a multiple of 64 bits.

There's always a price to be paid somewhere. My solution is extremely space-efficient, not too bad for performance, and terrible for complexity and code size. I use C++ templates to create a different class for numbers of each different size in bytes. The operators for addition, comparison, etc. are overridden to work on arbitrary-sized arrays of bytes, and the class is declared with `__attribute__ ((packed))` so that arrays of them will not be padded to get nice data alignment.

I went through many versions of C code, inline assembly, and compiler intrinsics over the years to try to get these to use well-performing code. Most recently, I found that the [clang](https://clang.llvm.org/) compiler can recognize a specific code pattern as being a multi-precision add or subtract. So the routine for the `+=` operator looks like this, where `n` is an array of bytes, `b` is the template parameter which is the length of `n`, and `U64`, `U32` and `U16` are macros which cast the argument to be a 64/32/16-bit pointer and dereference it:

```
inline varnum<b>& operator += (const varnum<b> &x)
{
    int64_t carry=0;

    // Add all the full 8-byte chunks of x to this. The code uses 128-bit math to add the two
    // sources plus carry-in and then extracts the carry-out from the upper 64 bits of the result,
    // but the compiler should turn this into an efficient sequence of add-carry instructions.
    for (int i=0; i<=b-8; i+=8) {
        int128_t temp = (int128_t)carry + U64(n+i) + U64(x.n+i);
        U64(n+i) = temp;
        carry = temp >> 64;
    }

    // Add any remaining upper bytes using 4/2/1-byte instructions
    if (b & 0x4) {
        int64_t temp = (int64_t)carry + U32(n+(b&(~0x7))) + U32(x.n+(b&(~0x7)));
        U32(n+(b&(~0x7))) = temp;
        carry = temp >> 32;
    }
    if (b & 0x2) {
        int32_t temp = (int32_t)carry + U16(n+(b&(~0x3))) + U16(x.n+(b&(~0x3)));
        U16(n+(b&(~0x3))) = temp;
        carry = temp >> 16;
    }
    if (b & 0x1)
        n[b-1] = carry + n[b-1] + x.n[b-1];

    return *this;
}
```

Since `b` is known at compile time, this produces the optimal sequence of add-carry instructions: as many 64-bit add-carrys as needed, and maybe 4/2/1-byte add-carrys to finish off the bytes at the top, if needed depending on the size of the number.

Similar routines implement other basic operations like assignment, arithmetic, and comparison, with special cases for some common things like comparison to zero and comparison to x +/- 1. For div and mod by 3, I just convert to a GMP integer and use the library routines. 

### Storing ranges

Since the sequence naturally contains many sequences of sequential numbers, it's much more efficent to store just the endpoints of ranges. A good structure for quickly adding and removing ordered things is a [red-black tree](https://en.wikipedia.org/wiki/Red%E2%80%93black_tree). But a fast implementation of a red-black tree wants to store a bunch of stuff in each node, including pointers to its left and right children, parent, and next/previous nodes, which is a lot of storage overhead.

One trick to cut the size of the pointers in half is to allocate a large single storage array up front, and use 32-bit indices into it, instead of actual 64-bit pointers. But it's still a lot of overhead. To amortize that cost, each node stores not just one range, but an array of ranges. The size of the array varies with the size of the numbers being stored in the tree, but is kept between 100 and 200. (The reason for not having a constant array size is explained below.)

The basic `range_tree` is templated on the size of the `varnum` numbers in its ranges, and supports several key operations:
- Adding a range: The new range may connect or overlap one or more ranges within a node, and/or span multiple nodes (which leads to _so many special cases_). If the new range combines with existing ranges then the node may end up with fewer valid ranges than it had before. If a new range needs to be inserted then the tail of the array is shifted over to make room; if the array is full then the last range is moved to a newly inserted node, making room to insert the new range.
- Querying whether a particular number is present in any of the ranges.
- Find the closest number lower than N which is taken. This is the check for case #1 in the fast computation.
- Find the closest number lower than N which is _not_ taken and is a multiple of 3 away from N. This is the check for case #2.
- Garbage collection: over time the tree gets cluttered with underutilized nodes, which occur when ranges coalesce or when new nodes containing a single range are inserted. When the ratio of invalid ranges gets too high, the tree repacks all ranges into full nodes, and recycles all extra nodes back into a freelist. It then shuffles the nodes in the underlying storage array so that they are located at sequential addresses, which helps with memory locality and paging (below).

Overall, the base `range_tree` gives highly space-efficient storage for ranges of a specific number size, with reasonable performance; using appropriate choices for array size and garbage collection thresholds, less than 1% of storage is used for anything other than the actual, non-zero data bytes.

### Variable-sized ranges

The next layer is the `var_range_tree`, which has an array of `range_tree` of each size. In fact, it's advantageous to limit the size of any individual tree (for garbage collection; see paging section below), so I keep one tree for every *bit* size. I set the maximum number size to 2048 bits, so I have 2048 trees of ranges.

All the math in the main algorithm is done using max-sized numbers, which are the inputs to the `var_range_tree` functions. It determines the actual size in bits of the input number and passes the operation on to the appropriate tree. For the search functions (e.g. closest lower taken number), an individual tree may return that its search fell off the lower end of the tree, in which case the same query is made of the next smaller tree. All the `var_range_tree` functions track a bitmap of which trees have been accessed and/or modified, which is periodically output and reset -- this is the source of the "paint-dripping" graphs above.

### OLD

The _persistence_ of a number is how many times you need to multiply
the digits together before you get to a single digit. Brady Haran and
Matt Parker made a fun [Numberphile](https://www.numberphile.com)
video about it: [What's special about
277777788888899?](https://www.youtube.com/watch?v=Wim9WJeDTHQ)

For example, start with 77:
- 7 * 7 = 49
- 4 * 9 = 36
- 3 * 6 = 18
- 1 * 8 = 8

So the persistence of 77 is 4.

What is the largest persistence a number can have? What is the
smallest number with persistence n? [A003001](http://oeis.org/A003001)
has some answers.

### Where to start

The first thing we notice is that any number which contains a 0 has
persistence 1 -- when we multiply the digits, we'll get 0. Similarly,
almost any number which _doesn't_ contain a 0 has persistence at least
2 -- when we multiply the digits we'll get something which is
definitely non-zero, which means we can go at least one more
step. (The only exception is numbers like 111111 or 221111 whose
digits multiply to give a one-digit but non-zero result.)

So numbers with persistence 2 aren't very interesting. There are
obviously infinitely many of them, and they're easy to find -- almost
any string of non-zero digits will do.

Also, there are lots of numbers which produce any given product of
digits -- you can rearrange the digits in any order, and add any
number of ones. For example, 73 produces 21, and so do 37, 731, 137,
7311111, etc.

To look for numbers with persistence 3 or more, instead of thinking
about the actual starting number, we can look at the result of the
first step: a number which we got by multiplying digits
together. These numbers are the product of numbers from 1 to 9, which
means they can only have prime factors 2, 3, 5, and 7. And if a number
has both 2 and 5 as factors, then it's a multiple of 10, which means
it has a zero, which means that the persistence process will be done
in the next step. So in fact we only need to look at numbers whose 
prime factors are in {2,3,7} or {3,5,7}.

### The big idea

So here's the general plan: we'll search all numbers of the form
7<sup>i</sup>\*3<sup>j</sup>\*2<sup>k</sup> or
7<sup>i</sup>\*3<sup>j</sup>\*5<sup>k</sup> up to some limit. We'll
check whether the number contains a 0, which means it's not
interesting -- we'll be done in the next step. In the unlikely event
that it _doesn't_ contain a 0, we'll actually calculate its
persistence.

But we're going to be getting into some pretty big numbers, so we will
need some tricks to speed up the multiplication of those prime factors
and the checking for zeros.

### Fast checking for zeros

Splitting off a single base-10 digit means division, which is slow. To
get the most out of our divisions, we can split off several digits at
once, and then check whether they contain a 0 using a pre-computed
lookup table. Keeping the table small enough to fit in the cache seems
to be good, so we'll use tables of all 4- and 5-digit numbers.

### Keeping it small

To avoid multiplying and dividing big numbers any more than we need,
we can start by just computing the low digits of the number. Probably
they will contain a 0, in which case we never need to look at the rest
of it.

Checking whether a number has any zeros proceeds in four steps, with
each calculation done using pre-computed powers of {2,3,5,7} of the
appropriate size:
- **mod 10<sup>9</sup>**: use 64-bit math to multiply 32-bit powers to
9 digits of accuracy. Split the result into lower 5 and upper 4
digits, and check for zeros using tables. This finds a zero about 55%
of the time.
- **mod 10<sup>19</sup>**: use 128-bit math to multiply 64-bit powers
to 19 digits of accuracy. We've already checked the low 9 digits so
check the upper digits in two 5-digit blocks. This finds a zero about
29% of the time.
- **mod 10<sup>76</sup>**: use GMP to compute 76 digits, and then
split into four 19-digit blocks. Each block is further split into
5/5/5/4 digits for the table check (except the first, which has
already been checked). This finds a zero about 15.5% of the time.
- **Full precision**: The checks above find a zero about 99.96% of the
time. If all those failed, just compute the whole thing using GMP and
scan it for zeros 19 digits at a time, and if necessary repeat on the
product of the digits to get the final persistence value.

### Results

It appears that there are exactly two numbers which are a product of
digits and have persistence 10 -- meaning there are two "interestingly
different" starting numbers with persistence 11 -- and that's the best
you can do.

Remember that we are searching numbers _P_ which are the product of
the digits of some other (larger) number. If one of those has a
persistence of N, that means there is an infinite group of numbers
with persistence N+1 whose digits multiply together to give
_P_. Anything in our search with a persistence of 2 or more is
interesting.

Let p(n) be the product of the digits of n, and P(n) be the
multiplicative persistence of n. The following are true of all p(n) <
10<sup>20000</sup>:

- There are 2 p(n) with P(p(n))=10. The largest is 2<sup>4</sup> *
3<sup>20</sup> * 7<sup>5</sup> (15 digits) The other one is
2<sup>19</sup> * 3<sup>4</sup> * 7<sup>6</sup>.
- There are 2 p(n) with P(p(n))=9. The largest is 2<sup>33</sup> *
3<sup>3</sup> (12 digits). The other one is 2<sup>12</sup> *
3<sup>7</sup> * 7<sup>2</sup>.
- There are 5 p(n) with P(p(n))=8. The largest is 2<sup>9</sup> *
3<sup>5</sup> * 7<sup>8</sup> (12 digits).
- There are 8 p(n) with P(p(n))=7. The largest is 2<sup>24</sup> *
3<sup>18</sup> (16 digits).
- There are 12 p(n) with P(p(n))=6. The largest is 2<sup>24</sup> *
3<sup>6</sup> * 7<sup>6</sup> (16 digits).
- There are 41 p(n) with P(p(n))=5. The largest is 2<sup>35</sup> *
3<sup>2</sup> * 7<sup>6</sup> (17 digits).
- There are 142 p(n) with P(p(n))=4. The largest is 2<sup>59</sup> *
3<sup>5</sup> * 7<sup>2</sup> (22 digits).
- There are 387 p(n) with P(p(n))=3. The largest is 2<sup>4</sup> *
3<sup>17</sup> * 7<sup>38</sup> (42 digits).
- There are 11994 p(n) with P(p(n))=2. The largest is 2<sup>25</sup> *
3<sup>227</sup> * 7<sup>28</sup> (140 digits).

All p(n) between 10<sup>140</sup> and 10<sup>20000</sup> have a
persistence of 1, meaning they contain a 0 digit.

Here is [a list](all-persistence.txt) of all numbers up to
10<sup>20000</sup> which are a product of digits and have persistence >= 2.

Of course this search doesn't prove
that 11 is the maximum persistence, but the base-10 digits of
7<sup>i</sup>\*3<sup>j</sup>\*2<sup>k</sup> or
7<sup>i</sup>\*3<sup>j</sup>\*5<sup>k</sup> are effectively
random. There are 3.3 billion 20000-digit "candidates" which are a
product of digits, and the chances of any one of them not having a 0
is less than 1 in 10<sup>-915</sup>. With each additional digit, the
number of candidates grows by only a tiny fraction, but the chance of
not having a 0 is multiplied by 0.9. So it seems very unlikely that
there will be any more numbers with even persistence 3.
