---
layout: post
title:  "Multiplicative Persistence"
date:   2019-09-27
---
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
