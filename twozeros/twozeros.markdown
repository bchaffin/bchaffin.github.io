---
layout: post
title:  "Zeros in Powers of 2"
date:   2019-10-01
---
This might be the most mathematically pointless project in my whole
list. Algorithmically, though, it got pretty interesting. And it's one
of my favorite examples of OEIS serendipity -- I stumbled across this
sequence because I mistyped an A-number into my search.

At the 13th [Gathering for Gardner](https://www.gathering4gardner.org)
conference I gave a [6-minute
talk](https://www.youtube.com/watch?v=7ySMrzNwrYQ) on this project.

The question is simple: what is the first power of 2 containing N
consecutive zeros when it's written out in decimal? The answer (up to
19 zeros) is in [A006889](https://oeis.org/A006889).

For example, the first powers of 2 to have 1, 2, and 3, zeros are:
- 2<sup>10</sup> = 1<span style="color:red">**0**</span>24
- 2<sup>53</sup> = 9<span style="color:red">**00**</span>7199254740992
- 2<sup>242</sup> = 706738825911353731833319<span
style="color:red">**000**</span>2971674063309935587502475832486424805170479104

The sequence caught my eye because at the time the highest term was
18, which is 4,627,233,991. 2<sup>4627233991</sup> is a pretty big
number -- a quick check showed that it took more than an hour just to
convert  to decimal (it has 1,392,936,229 digits), so obviously
something clever must be going on.

### Not my ideas

[Clive Tooth](https://sites.google.com/site/clivetoothcopy/) had found
several of the previously known terms, using an algorithm he developed
along with Christian Bau. Some posts in Google Forums have an excellent
description of [Clive's
method](https://groups.google.com/d/msg/sci.math/n6_q9HN1SII/62YQ2E2Pt5MJ),
and [Christian's
improvements](https://groups.google.com/d/msg/sci.math/n6_q9HN1SII/Jyz7k5Iv-UkJ),
and some [more
details](https://groups.google.com/d/msg/sci.math/BS-T3wS1CZ0/Q9fwN0vqk0IJ).

I won't replicate all that detail here, but briefly, their algorithm
relies on two key observations:

First, converting binary to decimal is division, and division is hard
and slow. So instead of working in binary, we should work directly in
decimal (or something like it).

Second, doubling a number does not randomly scramble all of the
digits. A string of zeros will emerge gradually, growing and then
shrinking across many doublings. A doubling can create at most one
zero on the left, if the previous digit was a 5. And zeros on the
right will be destroyed about every log<sub>2</sub>10 = 3.3 doublings,
as the digits on the right carry out. For example:

- 115<span style="color:red">**000**</span>234
- 23<span style="color:red">**0000**</span>468 -- doubling produced a
zero because there was a 5 on the left 
- 46<span style="color:red">**0000**</span>936 -- still four zeros
- 92<span style="color:red">**000**</span>1872 -- digits to the right
carry out, consuming one zero 

So we don't need to inspect every single power of 2 -- it's enough to
check only periodically, as long as we can bound the number of zeros
we might have missed in the interval we skipped over.

After working through some details (described in the posts above),
they settled on working in base 10<sup>9</sup>, and multiplying by
2<sup>34</sup> at every step. This works out very nicely because
10<sup>9</sup> is just less than 2<sup>30</sup>, so the mutiplication
of one base-billion "digit" by 2<sup>34</sup> just fits within 64
bits, making efficient use of the processor's datapath.

If you check every 34th power of 2, you are guaranteed to see a string
of at least 11 zeros, if there was a string of 18 zeros in some power
that you skipped over. If there are 11 zeros spread across
base-10<sup>9</sup> digits, then one of those digits must be less than
10<sup>6</sup>, which occurs only 1 in 1000 times. So that makes for a
very quick check in the common case, and only rarely do we have to do
more work to count exactly how many zeros there are.

I was looking for a string of 19 zeros, and so could have tightened
the fast check to only need a closer look 1 in 10000 times. But that
doesn't gain any significant performance, so I left it as a check for
18 zeros.

### Surely I can do better

That's all very clever, but surely, I thought, there's room for
improvement. The obvious improvement to the algorithm is to do more
work with each step: use a larger base, or multiply by a larger power
of 2, or both. Maybe you could use base 10<sup>16-18</sup> and multiply by
around 2<sup>64-66</sup>, for an overall speedup of ~3.5x. But you
need to solve two problems: finding shorter strings of zeroes and
handling 128-bit results.

Finding small strings of zeroes can be done using a lookup table; some
experiments showed that as long as the table fits in the last level
cache it doesn't cost too much. But the 128-bit division is a
bear. Even with hand-coded inverse multiplication and a lot of
tweaking, the best I was able to do was on par with the simpler
algorithm (meaning 3.5x slower per iteration). Processors are just too
good at 64-bit math, and 10<sup>9</sup>*2<sup>34</sup> is just too
perfectly 64-bit efficient to beat.

I also tried a number of different binary-coded decimal (BCD)
encodings. Of course with BCD the div/mod is nearly free, but the
shift becomes expensive -- too expensive, as it turns out.

So that's it -- I was out of ideas, and couldn't find any fundamental
improvement to Clive and Christian's method.

### Make it parallel

So now we have a giant array of base-billion digits, and the at the
core of the algorithm is a loop which runs over that whole array,
multiplying each digit by 2<sup>34</sup>, adding in the carry from the
previous digit, and doing div/mod by 10<sup>9</sup> to get the new
digit and the carry out.

Propagating the carry from one digit to the next means that this whole
loop forms a giant serial dependency chain, which is really bad for
performance on a modern processor which excels at doing several
independent things at the same time. To get around this, we can
process the digits not in sequential order: if the array has 100
digits, we might do digit 0, then 30, then 70, then 1, 31, 72, 2,
32, 72, etc. The calculation of digits 0, 30, and 70 don't directly
depend on each other, so the processor can do them all at the same
time (or nearly), and by the time it moves on to the next group, the
results from the first group are available.

In practice it turns out to be faster to actually interleave the
elements of the array, which keeps all of our memory references
sequential. So instead of the sequential array elements storing digits
0,1,2,3,4,5,... they store digits 0,30,70,1,31,71,.... This is worth
it even though it means that as the number gets bigger through
repeated multiplications, it grows a non-interleaved "head", and the
interleaving has to be periodically rebalanced.

There's a bit of work to handle the carries across the sections, but
overall the interleaving *doubles* performance running on a single
thread.

### Using more threads

To split the work into multiple pieces we'll want to be able to start
at some power of 2 and search a range up from there. That means
converting some giant power of 2 to base-billion at startup time,
which is not simple.

The GMP library has a really nice implementation of base conversion,
using recursive subdivision of the input number and inverse
multiplication on segments which are big enough to make calculating
the inverse powers worth it. Unfortunately it only supports up to base 255.
Of course we could just convert to base 10 first and then to base
10<sup>9</sup>, but that does a lot more division than necessary, and
then undoes it to combine the base-10 digits into
base-billion. Instead I hacked the GMP source to output 32-bit digits
and support base 10<sup>9</sup>. The conversion of
2<sup>10000000000</sup> takes about an hour.

Normally I use multiple cores by just running many copies of the same
program. But an hour is a pretty significant startup cost, so for this
project I used the `pthreads` library to spawn multiple execution
threads. (Which was just as difficult and deadlock-filled as I
feared.) The master thread assigns each worker a section of the array
to process. The workers do the interleaved multiplication, and record
places where there is (or might be) a run of 11 zeros, and also the
carry out of each section. After all workers complete, the master
propagates carries across the thread-block boundaries and from one
interleaved section to the next, finishes the multiplication on any
extra non-interleaved "head" section, and then checks all the
locations flagged by the workers as possibly having an interesting
number of zeros to see if there was a run of 18 nearby.

### Results

Clive measured his performance in picoseconds per digit, with digit
here meaning base-10 digits if the algorithm were actually considering
every power of 2. This is a good metric because it measures a sort of
"delivered performance" and allows comparison of both algorithmic
improvements and code optimization.

Running on a Skylake 18-core machine, this code took about 0.33
picoseconds per digit. Another way of measuring it is that at
2<sup>4600000000</sup>, it can multiply the number by 2<sup>34</sup>
and check for strings of zeros about 60 times a second.

After running for about a month and a half on several older machines,
it found the first power of 2 which has 19 consecutive zeros:

## **2<sup>11,089,076,233</sup>**

It has 3,338,144,571 digits, and contains the string
...66079358594929009959<span
style="color:red">**0000000000000000000**</span>912877187068429640146...
about 5% of the way in.

So, that was not very useful. But it was fun. :)
